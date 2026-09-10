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

`kong.yml` perdeu as rotas `rest-v1`, `graphql-v1` e `functions-v1` — nenhum
vestígio do que decidimos não ter.

**Ficam, e por quê:** `imgproxy` (o `storage` declara `depends_on` nele) e
`volumes/db/_supabase.sql` (o `supavisor` conecta no banco `_supabase`; apagar
esse init quebra o pooler).

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

**A correção é no SERVIDOR, não neste repositório:** apagar o volume `db-config`
e subir de novo. Ele só guarda a chave do pgsodium, que é regerada.

```sh
docker volume ls | grep db-config      # achar o nome com o prefixo do projeto
docker volume rm <projeto>_db-config
```

Não use `docker compose down -v`: aquilo apaga o `db-data` junto, e com ele o
banco. **Regra que fica:** trocou a tag da imagem do `db`, apague o `db-config`.

### 2. Postgres 17 não abre um data dir de Postgres 15

Medido: o container sai com código 1 e `FATAL: database files are incompatible
with server`. Ou o data dir nasce vazio, ou roda-se `utils/upgrade-pg17.sh`
antes — este preserva os dados, e um volume vazio não.

### 3. PGDATA em bind mount pode pular os init scripts

Por isso o PGDATA é volume **nomeado** (`db-data`). Medido no macOS com bind
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
