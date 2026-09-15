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

## O banco na rede do EasyPanel — o que essa exceção custa

Desde 2026-09-10 o `pxz-db` tem um **segundo pé na rede do EasyPanel**, como o
`pxz-kong`, porque a aplicação (`clinic`) precisa alcançá-lo. É uma exceção
consciente ao isolamento da seção acima, e o que ela troca precisa estar escrito:

- **O prefixo `pxz-` protege contra COLISÃO de nome, não contra ALCANCE.** Com o
  segundo pé, qualquer container da rede compartilhada resolve `pxz-db` e abre
  conexão na 5432. O que separa é **credencial**, e só ela.
- **A senha do `pxz_provisioner` passa a valer mais**, porque ele tem `bypassrls`
  e enxerga todas as clínicas. O `deploy.sql` cria os dois papéis com a MESMA
  senha e manda trocar a dele logo depois — nesta topologia isso deixa de ser
  zelo e vira obrigação.
- **A alternativa é melhor, e depende do EasyPanel:** pôr o serviço `clinic` na
  `pxz-internal` em vez de tirar o banco de lá. Aí ninguém mais na rede
  compartilhada enxerga a 5432. Se o painel deixar a aplicação entrar numa rede
  declarada por outro compose, é esse o caminho — e o segundo pé do `pxz-db` sai.

**Ficam, e por quê:** `imgproxy` (o `storage` declara `depends_on` nele) e
`volumes/db/_supabase.sql` (o `supavisor` conecta no banco `_supabase`; apagar
esse init quebra o pooler).

**E `volumes/db/webhooks.sql` fica mesmo sem usarmos webhook.** Medido em
2026-09-09, por acidente: é ele que CRIA o papel `supabase_functions_admin`, que
o `roles.sql` altera logo depois — daí `98-` antes de `99-`. Sem ele o init
aborta com `role "supabase_functions_admin" does not exist`, o container sai com
código 3 e o banco nunca nasce. Este repositório existe para tirar peça; esta
não sai.

**Desde 2026-09-10 sobram TRÊS: `pxz-db pxz-auth pxz-kong`.** Ver abaixo.

## Só o GoTrue: a superfície mínima (2026-09-10)

**Revisto em 2026-09-12: o Studio e o pg-meta VOLTARAM, com um portão** — ver a seção
"O Studio de volta". O que segue é o registro de como a superfície ficou mínima, e continua
valendo para tudo que não é Studio.

O cliente fechou a decisão: **o banco de prontuário NÃO se move** — fica no
PostgreSQL próprio —, e do Supabase usamos **só o Auth**. O Postgres deste pacote
guarda apenas o schema `auth` do GoTrue.

Consequência: `pxz-studio`, `pxz-meta`, `pxz-storage`, `pxz-imgproxy`,
`pxz-realtime` e `pxz-supavisor` **saíram do arquivo**. Nenhum deles é chamado
pela aplicação: mídia é MinIO por decisão do CLAUDE.md, não usamos websocket do
Supabase, e o pooler servia um banco que agora só o GoTrue usa.

**O GoTrue não precisa de irmãos — precisa do banco, e nada mais.** Medido em
2026-09-10: com o Studio PARADO, `GET /auth/v1/health` pelo Kong devolveu `200` e
`{"version":"v2.186.0","name":"GoTrue"}`. O único acoplamento era de compose, não
de runtime: `pxz-kong` tinha `depends_on: pxz-studio`, e um `up` arrastava o
Studio junto. Saiu.

**O `kong.yml` ficou com SETE rotas, todas de `/auth/v1`** — a mesma regra dos
desvios 3 e 4: nenhum vestígio do que decidimos não ter. Saíram `realtime-v1-ws`,
`realtime-v1-rest`, `storage-v1`, `meta`, `mcp`, `mcp-blocker`,
`well-known-oauth` (descoberta de OAuth, que sem provedor externo não serve) e
`dashboard`.

**O Studio deixou de existir por CONSTRUÇÃO, não por restrição.** Não há rota
para ele, e o consumer `DASHBOARD` com o `basicauth` saiu junto — sem rota, aquela
senha não guardava nada. Medido: `GET /` no Kong responde **404**.

Isto também fecha, de lado, o achado de 2026-09-10 de que o Studio conectava como
`supabase_admin` — superusuário. Com o Supabase servindo só o GoTrue, o banco dele
não tem prontuário; e sem rota, ninguém chega ao painel de qualquer forma.

**Os init scripts do banco FICAM, todos.** `realtime.sql`, `pooler.sql` e
`_supabase.sql` são de serviços que saíram, e mesmo assim não os removi: eles
rodam uma vez, criam objeto que ninguém lê, e não são superfície — não escutam
porta nem resolvem nome. Mexer neles é mexer no único caminho deste repositório
que já quebrou duas vezes (ver os três modos de falha abaixo), e o `webhooks.sql`
é justamente um deles: **é ele que CRIA o papel que o `roles.sql` altera**.

## A saída do pxz-auth, e por que ela NÃO é a rede compartilhada

O GoTrue precisa alcançar o servidor de e-mail — sem isso não há convite nem
recuperação de senha —, e a `pxz-internal` é `internal: true`: **não tem rota
para fora**. O pedido foi "um segundo pé na `default`, como o kong e o db".

**Entregamos a SAÍDA sem a ENTRADA**, numa rede dedicada (`pxz-saida`), e a
diferença não é estética. A `default` é a rede compartilhada do EasyPanel: com o
`pxz-auth` nela, qualquer container do servidor passaria a resolver `pxz-auth` e
a falar direto com a porta 9999 — **por fora do Kong**, que é justamente quem
exige a `apikey` nas rotas de `/auth/v1`. Adivinhação de senha contra o GoTrue
deixaria de passar pelo gateway.

**Medido em 2026-09-10, com controle:**

| de onde | `smtp.gmail.com:587` | `pxz-auth` é resolvível? |
|---|---|---|
| `pxz-saida` (a rede nova) | **alcança** | — |
| `pxz-internal` | não alcança | — |
| rede compartilhada, com o auth SÓ na saída | — | **não resolve** |
| rede compartilhada, com o auth NELA (controle) | — | resolve — é o que a `default` faria |

E a pilha real subiu com isso: os três `healthy`, e `/auth/v1/health` pelo Kong
continua `200`.

**Vale para o próximo serviço que precisar da internet:** a pergunta não é "ele
precisa de rede?", é "ele precisa FALAR ou precisa SER FALADO?". As duas têm
respostas diferentes, e só uma costuma ser o pedido.

## O Studio de volta (2026-09-12), e o portão que o acompanha

**Decisão do Pedro, com o risco ACEITO:** o Studio volta a ser servido na web pelo Kong,
porque o operador precisa OLHAR o schema `auth` do GoTrue e o único caminho era o console
do EasyPanel. O que torna isso aceitável hoje é uma coisa só, e ela está medida: **o
`pxz-db` tem apenas o schema `auth` — nenhum paciente** (zero tabelas em `app`, informado
pelo operador em 2026-09-10). Superusuário exposto alcança só os logins.

**O portão, na palavra dele, e está também no `CLAUDE.md` do prontuario-xz
("Infraestrutura"):** *Studio do Supabase aberto na web com pg-meta superusuário — decisão
do Pedro em 12/09, aceitável só enquanto o pxz-db tem apenas o schema auth. Fechar
obrigatoriamente antes de qualquer dado de prontuário migrar para este Postgres (restrição
por IP ou pg-meta com papel não-superusuário). Reabrir depois disso é vazamento de todos os
tenants.*

### O que voltou, e o que não voltou

| peça | estado | motivo |
|---|---|---|
| `pxz-studio`, `pxz-meta` | voltaram, **só na `pxz-internal`** | quem chega neles chega pelo Kong; nenhum outro projeto do servidor resolve os nomes |
| consumer `DASHBOARD` + `basicauth_credentials` | voltou | é o que guarda a rota `/` |
| rota `dashboard` (`/` → Studio, basic-auth) | voltou — **ABRE** | o pedido |
| rota `meta` (`/pg/` → pg-meta, key-auth + ACL `admin`) | voltou — **ABRE** | o pedido; só a `SERVICE_ROLE_KEY` passa |
| rotas `mcp-blocker` (`/api/mcp`) e `mcp` (`/mcp`), em 403 | voltaram — **FECHAM** | sem elas o catch-all de `/` alcançaria o servidor MCP do Studio, que executa SQL pelo pg-meta, atrás da MESMA senha. O modelo fecha os dois; aqui fica igual |
| `depends_on: pxz-studio` no Kong | **não voltou** | o gateway do LOGIN não pode esperar o Studio ficar `healthy` — é a cadeia `analytics -> studio -> kong` do desvio 1, de novo. O Kong resolve `pxz-studio` na requisição (`KONG_DNS_NOT_FOUND_TTL: 1`) |

As variáveis obrigatórias vão de 27 para **34** — as sete que o `AMBIENTE.md` mandava
apagar voltam, e a `DASHBOARD_PASSWORD` mudou de natureza. Ver `AMBIENTE.md`.

### Medido no ensaio local, com controle

Pilha inteira numa cópia descartável (`docker compose -p pxz-ensaio`), os cinco `healthy`,
sondas de um container na rede compartilhada contra `pxz-kong:8000`. Derrubada com `down -v`
no fim; nada foi escrito no servidor.

| sonda | resultado | o que prova |
|---|---|---|
| `GET /` sem credencial | **401** | a rota exige senha |
| `GET /` senha errada (o `Authorization` enviado foi conferido com `-v`) | **401** | controle negativo |
| `GET /` usuário errado, senha certa | **401** | segundo controle negativo |
| `GET /` senha certa | **307** → `/project/default` | controle positivo: o Studio está lá |
| `GET /pg/tables` sem `apikey` | **401** | key-auth |
| `GET /pg/tables` com a `anon` | **403** | ACL só `admin` |
| `GET /pg/tables` com a `service_role` | **200** | controle positivo |
| `POST /pg/query` `select current_user, usesuper` | **`supabase_admin`, `usesuper: true`** | **a frase do portão não é hipótese** |
| `GET /api/mcp` e `GET /mcp` com a senha certa | **403** | as duas rotas que FECHAM fecham |
| `GET /auth/v1/health` com a `anon` | **200** | o login continua de pé com o Studio ao lado |
| DNS da rede compartilhada | `pxz-kong` e `pxz-db` resolvem; `pxz-studio`, `pxz-meta` e `pxz-auth` **não** | o isolamento da seção do incidente continua igual |

### Como o portão se fecha, quando chegar a hora

Dois caminhos, e qualquer um serve; o segundo é o definitivo:

1. **`ip-restriction` na rota `dashboard`** do `kong.yml` — o plugin já está em
   `KONG_PLUGINS`. Fecha o Studio a quem não está na lista; o pg-meta continua superusuário.
2. **pg-meta sem superusuário**: `PG_META_DB_USER` no `pxz-meta` e `POSTGRES_USER_READ_WRITE`
   no Studio apontando para um papel criado para isso — Supabase, *Self-Hosting → Remove
   superuser access*. Fecha a causa, não o acesso.

**A senha.** `DASHBOARD_PASSWORD` é agora senha de superusuário do banco na internet. Hex, 32
caracteres no mínimo, nunca a do modelo, nunca reaproveitada — e **ninguém deste repositório
alcança o Ambiente para conferir a que está lá**. O passo de aplicar manda gerar uma nova
antes de colar o compose, em vez de confiar na que ficou.

### 14 e 15/09: o Studio esteve na web com credencial ADIVINHÁVEL

**O que aconteceu.** O compose que estava no servidor **não tinha `DASHBOARD_USERNAME` e
`DASHBOARD_PASSWORD` no serviço `pxz-kong`** — o `3e46937` tem as duas, mas não tinha sido
empurrado, e o que chegou ao EasyPanel foi outra coisa (mesclado à mão, ou o compose de
`a53d7a4`). Quando a rota do Studio entrou no ar, o entrypoint do Kong — o do modelo, intocado
desde `7de0108` — procurou as duas no ambiente do container, não achou, e **deixou o
placeholder no lugar**: a credencial do `basic-auth` virou usuário `$DASHBOARD_USERNAME` com
senha `$DASHBOARD_PASSWORD`. `kong config parse` aceita isso sem uma linha de aviso.

**Medido em 2026-09-14, com a pilha inteira e o serviço do Kong sem as duas linhas:**

| `GET /` pelo Kong | resultado |
|---|---|
| sem senha | 401 |
| com o usuário e a senha REAIS do Ambiente | 401 |
| digitando `$DASHBOARD_USERNAME` e `$DASHBOARD_PASSWORD` | **307 — entra** |

Não é "trancado com a chave perdida": é trancado com uma chave que está escrita na porta.

**Os três estados da variável, e só um deles é silencioso:**

| no container | credencial gerada | `kong config parse` | sintoma |
|---|---|---|---|
| declarada e preenchida | usuário e senha reais | passa | o Studio abre com a senha |
| declarada, vazia no Ambiente | `''` / `''` | **falha**: "length must be at least 1" | o Kong não sobe — alto |
| **não declarada no serviço** | os dois NOMES, literais | passa | pede senha, a certa não entra, a literal entra |

**O que fecha, e a ordem:** o `kong.yml` novo no bind mount do host ANTES (senão o deploy volta
ao 404); as duas variáveis no Ambiente (senão é a linha do meio, e o login cai junto); o compose
**inteiro** de um commit empurrado colado no EasyPanel; deploy. Depois, dois números no console
do `pxz-kong` e três de fora:

```sh
env | grep -c '^DASHBOARD_'                                 # 2
grep -c '\$[A-Z_][A-Z_0-9]*' "$KONG_DECLARATIVE_CONFIG"    # 0 — NENHUM placeholder sobrou
```

`GET /` sem senha 401; com os dois NOMES **401** (é esta a prova de que a porta fechou); com a
senha real 307.

**Três regras que ficam:**

1. **Compose no servidor é o ARQUIVO INTEIRO de um commit empurrado**, colado do GitHub pelo
   hash. Mesclar à mão foi o que deixou duas linhas para trás, e duas linhas foram uma
   credencial pública.
2. **Substituição de variável se confere pela AUSÊNCIA de qualquer `$NOME` no resultado**, não
   pela presença do que se lembrou de olhar. O bloco de console de 2026-09-14 conferia só
   `LUA_`, passou no ensaio porque o `.env` local tinha as variáveis, e no servidor relatou
   sucesso sobre uma credencial adivinhável. O controle certo está no bloco acima.
3. **Não use `/proc/1/environ` para ver o ambiente do Kong**: o nginx reescreve o ambiente do
   processo 1, e ele mostra zero até para variável que o container recebeu (medido). `env` no
   console do container é o instrumento certo.

**O que NÃO foi feito, e é a próxima defesa:** o entrypoint continua não parando em placeholder
não resolvido. Uma guarda de seis linhas depois do `awk` — contar `$NOME` no arquivo gerado e
sair com a lista — tornaria este estado impossível de repetir em silêncio. Custa um SEGUNDO
arquivo de bind mount a levar ao host a cada mudança; a alternativa é os dois arquivos
(`kong.yml` e o entrypoint) passarem a viajar DENTRO do compose, por `configs:`. Decisão para
quando o servidor estiver estável.

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

Nenhum segredo mora neste repositório. As obrigatórias vão em **Ambiente**, no serviço do
EasyPanel — a contagem e a lista vivem em `AMBIENTE.md`, e só lá.

## Não empacotado, de propósito

Os overlays alternativos do modelo (`caddy`, `envoy`, `nginx`, `rustfs`, `s3`,
`pg17`), mais `dev/` e `tests/`. O `envoy` reintroduz o `functions`, e o `pg17`
virou redundante — o 17 já está no arquivo principal. `volumes/` veio inteiro:
`volumes/logs/vector.yml` e `volumes/api/envoy/` estão sem uso.
