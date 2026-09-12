# Ambiente — as 34 variáveis

Vão em **Ambiente**, no serviço `supabase` do EasyPanel. Nenhuma neste repositório.
O EasyPanel interpola `$(PRIMARY_DOMAIN)` no próprio valor.

**Eram 46 até 2026-09-10.** A decisão de usar o Supabase **só como autenticação**
tirou seis serviços (`studio`, `meta`, `storage`, `imgproxy`, `realtime`,
`supavisor`) e, com eles, 19 variáveis — ficaram 27. **Em 2026-09-12 voltaram
SETE, com o Studio e o pg-meta** (decisão do Pedro, com o risco aceito e um
portão escrito no `CLAUDE.md` do prontuario-xz — ver README, "O Studio de volta").
A lista abaixo foi **medida** contra o `docker-compose.yml` (`${VAR}` sem default,
fora de comentário), não escrita de memória: 34.

## 6 SEGREDOS — gerar, nunca reaproveitar de exemplo

`utils/generate-keys.sh` gera o conjunto de uma vez (ele ainda imprime as antigas;
ignore as que não estão aqui).

| variável | como gerar |
|---|---|
| `POSTGRES_PASSWORD` | `openssl rand -base64 36` (sem `/` e `+` se o painel implicar) |
| `JWT_SECRET` | `openssl rand -hex 32` — mínimo 32 caracteres |
| `ANON_KEY` | JWT `role: anon` assinado com o `JWT_SECRET` — `utils/generate-keys.sh` |
| `SERVICE_ROLE_KEY` | JWT `role: service_role` com o mesmo segredo — idem |
| `PG_META_CRYPTO_KEY` | `openssl rand -base64 24` — é o que `generate-keys.sh` faz. **Voltou em 2026-09-12** |
| `DASHBOARD_PASSWORD` | `openssl rand -hex 16` **no mínimo** (32 caracteres). **Voltou em 2026-09-12** — ver abaixo, porque ela mudou de natureza |

`ANON_KEY` e `SERVICE_ROLE_KEY` **têm que ser assinadas com o `JWT_SECRET` desta
instalação**, e agora isso é mais do que uma regra de configuração: o Kong aceita
a chave como `apikey`, mas **o GoTrue valida a ASSINATURA do bearer**. Medido em
2026-09-10: uma chave que não é JWT passa pelo Kong e o GoTrue responde
`bad_jwt`. As duas coisas são conferidas em lugares diferentes.

**`SMTP_PASS` também é segredo**, quando existir. Ela está entre as não-secretas
abaixo porque hoje é vazia — ver o bloqueio.

### `DASHBOARD_PASSWORD` deixou de ser senha de painel

Ela é o `basic-auth` da rota `/` do Kong, e atrás dessa rota está o Studio falando
com um pg-meta que conecta como **`supabase_admin`, superusuário** — medido no
ensaio local de 2026-09-12 (`POST /pg/query` devolveu
`{"current_user":"supabase_admin","usesuper":true}`). Quem tem essa senha tem o
banco inteiro pela internet. Hoje o banco tem só o schema `auth`; é por isso, e só
por isso, que a rota voltou.

Três exigências, e o compose não confere nenhuma delas — quem confere é quem cola:

1. **Nunca a do modelo.** O `.env.example` do Supabase traz
   `this_password_is_insecure_and_should_be_updated` e usuário `supabase`. Se o
   valor que estava no Ambiente antes de 2026-09-10 for esse, ou tiver menos de
   32 caracteres, **troque ANTES de colar o compose** — a rota entra no ar no
   mesmo `up`.
2. **Única.** Não é a `POSTGRES_PASSWORD`, não é a da demo, não é a de nenhum outro
   serviço do servidor.
3. **Hexadecimal, sem `$`, `'` ou `\`.** O `kong-entrypoint.sh` substitui todo
   `$NOME` do `kong.yml` pelo ambiente, e o valor vai entre aspas simples no YAML:
   um `$`, um `'` ou uma barra invertida na senha corrompem a configuração em
   silêncio, e o Kong sobe com outra senha ou não sobe. `openssl rand -hex 16`
   não produz nenhum dos três.

`DASHBOARD_USERNAME` não é segredo, mas também não precisa ser `supabase`.

## 28 não-segredos — valor sugerido

### Banco e conexão
| variável | sugerido | nota |
|---|---|---|
| `POSTGRES_HOST` | `pxz-db` | nome do serviço. **Mudou em 2026-09-10**: era `db`, e nome genérico em rede compartilhada foi o incidente do Studio — ver README |
| `POSTGRES_PORT` | `5432` | |
| `POSTGRES_DB` | `postgres` | **é o banco do GoTrue, e só dele.** O prontuário fica no PostgreSQL próprio; ver a decisão no CLAUDE.md |

### Domínio
| variável | sugerido |
|---|---|
| `API_EXTERNAL_URL` | `https://$(PRIMARY_DOMAIN)` |
| `SITE_URL` | `https://$(PRIMARY_DOMAIN)` — o destino dos links de e-mail |
| `ADDITIONAL_REDIRECT_URLS` | *(vazio)* |
| `SUPABASE_PUBLIC_URL` | `https://$(PRIMARY_DOMAIN)` — **voltou em 2026-09-12** com o Studio, que a usa para montar as URLs que mostra |

### Studio — de volta em 2026-09-12
| variável | sugerido | nota |
|---|---|---|
| `DASHBOARD_USERNAME` | um nome que não seja `supabase` | o usuário do `basic-auth`; a senha está entre os segredos |
| `STUDIO_DEFAULT_ORGANIZATION` | `prontuario-xz` | só rótulo |
| `STUDIO_DEFAULT_PROJECT` | `auth` | só rótulo — e diz o que este banco é |
| `PGRST_DB_SCHEMAS` | `public` | o Studio exige a variável; **não há PostgREST** nesta pilha, então ela não expõe nada |

### Auth — o conjunto fechado, e é de propósito
| variável | sugerido | **difere do modelo** |
|---|---|---|
| `DISABLE_SIGNUP` | `true` | **sim** (modelo: `false`) — não há auto-cadastro; quem cria pessoa é `app.user_create` |
| `ENABLE_EMAIL_SIGNUP` | `true` | não — irrelevante com signup desligado |
| `ENABLE_EMAIL_AUTOCONFIRM` | `true` **enquanto não houver SMTP** | **sim** — ver o bloqueio |
| `ENABLE_PHONE_SIGNUP` | `false` | **sim** (modelo: `true`) — não há provedor de SMS |
| `ENABLE_PHONE_AUTOCONFIRM` | `false` | **sim** (modelo: `true`) |
| `ENABLE_ANONYMOUS_USERS` | `false` | não |
| `JWT_EXPIRY` | `3600` | não |

### SMTP — o BLOQUEIO de Auth, e o contorno é temporário
| variável | provisório | definitivo |
|---|---|---|
| `SMTP_HOST` | `smtp.invalid` | o servidor que o Renato escolher |
| `SMTP_PORT` | `587` | idem |
| `SMTP_USER` / `SMTP_PASS` | *(vazio)* | credencial do provedor |
| `SMTP_SENDER_NAME` | um nome da clínica | idem |
| `SMTP_ADMIN_EMAIL` | `nao-configurado@invalid` | um endereço real da equipe |
| `MAILER_URLPATHS_CONFIRMATION` / `_INVITE` / `_RECOVERY` / `_EMAIL_CHANGE` | `/auth/v1/verify` (as quatro) | iguais |

**`SMTP_PORT` vazio DERRUBA o GoTrue.** Medido em 2026-09-10 — não é aviso, é
saída fatal em laço:

```
fatal  Failed to load configuration: assigning GOTRUE_SMTP_PORT to Port:
       converting '' to type int
```

O `AMBIENTE.md` dizia "vazio" e registrava isto como *não medido*. Agora está
medido, e mudou de categoria: **SMTP é pré-requisito do serviço subir**, não um
detalhe da Etapa 4.

**O contorno, e o que ele NÃO entrega.** Com `SMTP_HOST=smtp.invalid`,
`SMTP_PORT=587` e `ENABLE_EMAIL_AUTOCONFIRM=true`, o GoTrue sobe `healthy` e
**o login por senha funciona** — medido de ponta a ponta em 2026-09-10: usuário
criado pela admin API e token emitido, sem uma linha de e-mail enviada. O que
não funciona, e nem parece que não funciona:

- **convite** — a pessoa nunca recebe o link;
- **recuperação de senha** — idem;
- **confirmação de e-mail** — contornada pelo autoconfirm, ou seja, ninguém
  prova que o endereço existe.

**Enquanto isso, quem cria conta é `app.user_create` (0035), com senha provisória
mostrada uma vez** — o caminho que já existe e não depende de e-mail.

## Com default no compose — só se quiser mudar
`OPENAI_API_KEY` (vazio), `PGRST_DB_MAX_ROWS` (1000), `PGRST_DB_EXTRA_SEARCH_PATH`
(`public`), `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SECRET_KEY`,
`ANON_KEY_ASYMMETRIC`, `SERVICE_ROLE_KEY_ASYMMETRIC` (as quatro vazias — são as
chaves opacas novas, não usadas aqui).

## O que SAIU em 2026-09-10 e NÃO voltou — 12
`SECRET_KEY_BASE`, `VAULT_ENC_KEY`, `S3_PROTOCOL_ACCESS_KEY_ID`,
`S3_PROTOCOL_ACCESS_KEY_SECRET`, `GLOBAL_S3_BUCKET`, `STORAGE_TENANT_ID`, `REGION`,
`IMGPROXY_AUTO_WEBP`, `POOLER_TENANT_ID`, `POOLER_DB_POOL_SIZE`,
`POOLER_DEFAULT_POOL_SIZE`, `POOLER_MAX_CLIENT_CONN`.

Podem ser apagadas do Ambiente. **As sete que este arquivo dizia para apagar e que
VOLTARAM em 2026-09-12** — `PG_META_CRYPTO_KEY`, `DASHBOARD_USERNAME`,
`DASHBOARD_PASSWORD`, `SUPABASE_PUBLIC_URL`, `STUDIO_DEFAULT_ORGANIZATION`,
`STUDIO_DEFAULT_PROJECT`, `PGRST_DB_SCHEMAS` — estão nas tabelas acima. Se foram
apagadas, entram de novo; se ficaram, a `DASHBOARD_PASSWORD` é a única que **não
pode** ser simplesmente mantida sem olhar — ver a seção dela.
