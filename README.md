# supabase-pxz

Supabase auto-hospedado para o **prontuario-xz**, no EasyPanel.

Origem: `easypanel-io/compose` @ ref `18-05-2026`, `/supabase/code`.
Os originais intocados dos dois arquivos que mexemos estão em `MODELO/`, para
`diff` a qualquer momento.

## Os cinco desvios em relação ao modelo

| # | desvio | motivo |
|---|---|---|
| 1 | `analytics` (Logflare) removido | ficava unhealthy e derrubava a pilha pela cadeia `analytics -> studio -> kong`; o kong é o gateway da porta 8000 |
| 2 | `vector` removido | só existia para despejar log no analytics, e montava o socket do Docker |
| 3 | `rest` (PostgREST) removido | decisão de segurança do CLAUDE.md: com ele publicado, o `withTenant` deixa de ser camada e o RLS vira a única linha de defesa |
| 4 | `functions` removido | `/functions/v1/` era a única rota de serviço sem key-auth nem ACL, e nada dependia dele |
| 5 | `db` em `supabase/postgres:17.6.1.084` | a regra do projeto é Postgres 17 em dev e em produção, sem exceção |

| 6 | todo serviço com prefixo `pxz-` | incidente de 2026-09-10 — ver abaixo |
| 7 | rede `pxz-internal` com `internal: true`, e só o `pxz-kong` fora dela | idem |

`kong.yml` perdeu as rotas `rest-v1`, `graphql-v1` e `functions-v1` — nenhum
vestígio do que decidimos não ter.

## O incidente de 2026-09-10, e por que os nomes têm prefixo

**Às 00:11 de 2026-09-10 o Studio aberto pelo NOSSO Kong mostrou o Supabase de
outro projeto do mesmo servidor EasyPanel** — o "TCC Flow", o CRM do Renato.
Dado de outro sistema servido pelo nosso gateway. O serviço foi parado.

**A causa é DNS, não autenticação.** Os projetos dividem rede no EasyPanel, e os
nomes de serviço do modelo são genéricos: `db`, `studio`, `kong`, `auth`, `meta`.
Numa rede compartilhada, `studio` resolve para *algum* container com esse nome —
e não há nada que garanta que seja o nosso. Nenhuma senha falhou: o Kong
perguntou por `studio` e a rede respondeu com o de outro projeto.

**Duas correções, e as duas são necessárias:**

1. **Nome próprio.** Os nove serviços passaram a `pxz-db`, `pxz-studio`,
   `pxz-kong`, `pxz-auth`, `pxz-meta`, `pxz-storage`, `pxz-realtime`,
   `pxz-imgproxy`, `pxz-supavisor`, e **toda** referência por nome foi junto: os
   `depends_on`, `STUDIO_PG_META_URL`, `SUPABASE_URL`, `IMGPROXY_URL`, o
   healthcheck do storage, os 25 upstreams do `kong.yml` e o fallback do
   `pooler.exs`. Medido depois: dentro da nossa rede, `db` e `studio` **não
   resolvem mais**, e `pxz-db` resolve.
2. **Rede fechada.** `pxz-internal` é `internal: true` e leva os nove; **só o
   `pxz-kong` tem um segundo pé na rede do EasyPanel**, porque é ele o gateway.
   Medido: container só na interna não alcança a internet, resolve os irmãos por
   nome, e de fora dela ninguém o enxerga.

**Nome próprio sem rede fechada não bastaria**, e é o que torna a segunda
correção obrigatória: prefixo evita a colisão acidental, mas o banco continuaria
alcançável de qualquer container do servidor por quem soubesse o nome. E rede
fechada sem nome próprio também não: dois projetos na mesma rede compartilhada
voltam a colidir.

**Um defeito HERDADO que apareceu no caminho, e era da mesma família.** O
`kong.yml` roteava `realtime-dev.supabase-realtime`, e **nem o modelo nem nós
temos `container_name`** — esse nome nunca resolveu na nossa pilha. Numa rede
compartilhada, um nome pendurado assim é exatamente o que encontra o container de
outro projeto. Agora é o alias `realtime-dev.pxz-realtime`, declarado no serviço:
o primeiro rótulo continua sendo `realtime-dev`, que é de onde o realtime tira o
tenant, e o resto é nosso.

**O que isso custa:** serviço que só vive na rede interna **não alcança a
internet**. Hoje nada precisa — SMTP está vazio e não há provedor externo de
login. **Na Etapa 4 isso morde**: para o GoTrue mandar e-mail, o `pxz-auth`
precisa de um segundo pé fora da interna, como o Kong tem.

**Conferir no EasyPanel:** o roteamento de domínio aponta para o serviço pelo
NOME. Era `kong`, agora é `pxz-kong`.

**Ficam, e por quê:** `imgproxy` (o `storage` declara `depends_on` nele) e
`volumes/db/_supabase.sql` (o `supavisor` conecta no banco `_supabase`; apagar
esse init quebra o pooler).

**E `volumes/db/webhooks.sql` fica mesmo sem usarmos webhook.** Medido em
2026-09-09, por acidente: é ele que CRIA o papel `supabase_functions_admin`, que
o `roles.sql` altera logo depois — daí `98-` antes de `99-`. Sem ele o init
aborta com `role "supabase_functions_admin" does not exist`, o container sai com
código 3 e o banco nunca nasce. Este repositório existe para tirar peça; esta
não sai.

Sobram nove serviços: `auth db imgproxy kong meta realtime storage studio supavisor`.

## Antes de implantar

São **três** modos de falha do `db`, e o log do container é o que os separa —
a correção difere em cada um.

### 1. `db-config` herdado de outra versão da imagem

**É a causa do laço de 2026-09-09 no servidor**, e foi reproduzida no Mac:

```
Success. You can now start the database server using: ...
LOG:   could not open configuration directory "/etc/postgresql-custom/conf.d": No such file or directory
FATAL: configuration file "/etc/postgresql/postgresql.conf" contains errors
```

O initdb conclui, o servidor sai com código 1, e o `restart: unless-stopped`
recomeça — laço. **Não é parâmetro recusado:** é um `include_dir` apontando para
um diretório que não existe. Medido nas duas imagens: `/etc/postgresql-custom`
tem `conf.d` na `17.6.1.084` e **não tem** na `15.8.1.085`, e a `postgresql.conf`
do 17 faz `include_dir` nele. Volume nomeado **não é repovoado** quando a imagem
muda de versão, então um `db-config` criado pelo 15 derruba o 17 para sempre.

O `docker-compose.pg17.yml` do modelo avisa disso — e é a única coisa além da tag
da imagem que ele traz: *"If starting fresh with a leftover db-config volume from
PG 15, you may see FATAL: invalid secret key. Either remove the old volume or fix
ownership."* Nós levamos a tag e deixamos o aviso para trás.

#### A correção, e ela tem dois caminhos

**Com acesso ao host** — apagar o volume e subir de novo. É o caminho limpo: não
deixa nada para trás. O `db-config` só guarda a chave do pgsodium, que é regerada.

```sh
docker volume ls | grep db-config      # achar o nome com o prefixo do projeto
docker volume rm <projeto>_db-config
```

Não use `docker compose down -v`: aquilo apaga o PGDATA junto, e com ele o banco.

**Sem acesso ao host — é o caso deste repositório, e é o que está no arquivo.**
Quando o painel não dá terminal na máquina, não há como apagar volume. A saída é
**renomear o volume no `docker-compose.yml`**: nome novo é volume novo, o `up` o
cria VAZIO, e o Docker o popula a partir da imagem — que é exatamente o que
faltava. O órfão continua no host, intocado e sem ser lido.

Desde 2026-09-09 os dois têm sufixo de versão: `db-config-pg17` e `db-data-pg17`.

O que isso custa, e não é zero:

- **O PGDATA nasce vazio: o banco perde o que houver no volume antigo.** Aqui não
  havia o que perder — a pilha nunca subiu inteira, e o único `initdb` que
  concluiu foi o do laço. **Em servidor com dado, renomear o PGDATA é destrutivo**,
  e aí o caminho é o de cima, ou `utils/upgrade-pg17.sh`.
- **Os volumes órfãos ficam ocupando disco** até alguém alcançar o host. São dois.
- **É bom que o PGDATA também nasça vazio**, e por um motivo que não é o do laço:
  `roles.sql` roda **só no init**, e é ele que aplica o `POSTGRES_PASSWORD` aos
  papéis do banco. Com as chaves rotacionadas em 2026-09-09, um cluster
  inicializado com a senha velha responderia `28P01` a todo serviço, com o `.env`
  novo e correto. Init novo remove essa ambiguidade.

**Regra que fica:** trocou a tag da imagem do `db`, ou apague o `db-config` no
host, ou mude o sufixo dos dois volumes no compose. Não fazer nem um nem outro é
o laço deste capítulo, de novo.

### 2. Postgres 17 não abre um data dir de Postgres 15

Medido: o container sai com código 1 e `FATAL: database files are incompatible
with server`. Ou o data dir nasce vazio, ou roda-se `utils/upgrade-pg17.sh`
antes — este preserva os dados, e um volume vazio não.

### 3. PGDATA em bind mount pode pular os init scripts

Por isso o PGDATA é volume **nomeado** (`db-data-pg17`). Medido no macOS com bind
mount: `PostgreSQL Database directory appears to contain a database; Skipping
initialization`, o `roles.sql` não roda, e o banco fica com **1 papel em vez de
13** — todo serviço falha com `28P01`. Os seis init scripts continuam bind
mount; só o PGDATA mudou.

**Correção do registro:** o `unhealthy` em ~7 s de 2026-09-09 foi atribuído a
este modo, e a troca para volume nomeado foi feita por causa dele. Não era —
era o modo 1. A troca fica, porque o modo 3 é real e está medido, mas ela não
resolveu nada naquele dia. Diagnóstico atribuído sem ler o log do container é o
que produziu um commit que não consertou o que se propunha a consertar.

### Por que `log_min_messages=warning`, e não `fatal`

O modelo usa `fatal`, para o Realtime não encher o log com as consultas de
polling. **Medido: `fatal` suprime a linha `LOG` que NOMEIA o arquivo que
falta**, e sobra só `contains errors`, que não diagnostica nada — foi
exatamente o que cegou o servidor. Os dois lados, mesmo defeito:

| `log_min_messages` | o que o log mostra |
|---|---|
| `fatal` (modelo) | só `FATAL: ... contains errors` |
| `warning` (nosso) | `LOG: could not open configuration directory ...` **e** o `FATAL` |

O preço é o log voltar a ter as consultas do Realtime. **Não medido:** o volume
desse ruído — o Realtime não subiu na reprodução, que foi só do `db`.

## Variáveis

Nenhum segredo mora neste repositório. As 46 obrigatórias vão em **Ambiente**,
no serviço do EasyPanel — ver `AMBIENTE.md`.

## Não empacotado, de propósito

Os overlays alternativos do modelo (`caddy`, `envoy`, `nginx`, `rustfs`, `s3`,
`pg17`), mais `dev/` e `tests/`. O `envoy` reintroduz o `functions`, e o `pg17`
virou redundante — o 17 já está no arquivo principal. `volumes/` veio inteiro:
`volumes/logs/vector.yml` e `volumes/api/envoy/` estão sem uso.
