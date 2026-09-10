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

**Postgres 17 não abre um data dir de Postgres 15.** Medido: o container sai com
código 1 e `FATAL: database files are incompatible with server`. Ou o data dir
nasce vazio, ou roda-se `utils/upgrade-pg17.sh` antes.

**O PGDATA é volume NOMEADO (`db-data`), não bind mount.** Medido no macOS com
bind mount: `PostgreSQL Database directory appears to contain a database;
Skipping initialization`, o `roles.sql` não roda, e o banco fica com **1 papel em
vez de 13** — todo serviço falha com `28P01`. Com volume nomeado, inicializa
correto. Os seis init scripts continuam bind mount; só o PGDATA mudou.

Em 2026-09-09 o `db` subiu **unhealthy em ~7 s no servidor**, com o `.env` já
criado — relato do operador, que atribuiu ao PGDATA. A troca é dessa data.
**Não medido:** que ela resolve. Ninguém deste repositório alcança o servidor.

**E o log do container é o que separa os dois modos de falha**, porque a correção
difere. `database files are incompatible with server` é data dir do 15 aberto
pelo 17, e aí o caminho é `utils/upgrade-pg17.sh` **com os dados preservados** —
volume nomeado nasce vazio, então o que estiver em `volumes/db/data` deixa de ser
lido (não é apagado, fica órfão). Numa instalação nova não há o que preservar.

## Variáveis

Nenhum segredo mora neste repositório. As 46 obrigatórias vão em **Ambiente**,
no serviço do EasyPanel — ver `AMBIENTE.md`.

## Não empacotado, de propósito

Os overlays alternativos do modelo (`caddy`, `envoy`, `nginx`, `rustfs`, `s3`,
`pg17`), mais `dev/` e `tests/`. O `envoy` reintroduz o `functions`, e o `pg17`
virou redundante — o 17 já está no arquivo principal. `volumes/` veio inteiro:
`volumes/logs/vector.yml` e `volumes/api/envoy/` estão sem uso.
