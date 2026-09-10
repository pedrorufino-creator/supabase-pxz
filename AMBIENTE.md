# Ambiente — as 46 variáveis

Vão em **Ambiente**, no serviço `supabase` do EasyPanel. Nenhuma neste repositório.
O EasyPanel interpola `$(PRIMARY_DOMAIN)` no próprio valor.

## 10 SEGREDOS — gerar, nunca reaproveitar de exemplo

`utils/generate-keys.sh` gera o conjunto de JWT de uma vez.

| variável | como gerar |
|---|---|
| `POSTGRES_PASSWORD` | `openssl rand -base64 36` (sem `/` e `+` se o painel implicar) |
| `JWT_SECRET` | `openssl rand -hex 32` — mínimo 32 caracteres |
| `ANON_KEY` | JWT `role: anon` assinado com o `JWT_SECRET` — `utils/generate-keys.sh` |
| `SERVICE_ROLE_KEY` | JWT `role: service_role` com o mesmo segredo — idem |
| `SECRET_KEY_BASE` | `openssl rand -base64 48` — realtime e supavisor |
| `VAULT_ENC_KEY` | `openssl rand -hex 16` — exatamente 32 caracteres |
| `PG_META_CRYPTO_KEY` | `openssl rand -hex 32` |
| `DASHBOARD_PASSWORD` | `openssl rand -base64 24` — é o que separa a internet do Studio |
| `S3_PROTOCOL_ACCESS_KEY_ID` | `openssl rand -hex 16` |
| `S3_PROTOCOL_ACCESS_KEY_SECRET` | `openssl rand -hex 32` |

`ANON_KEY` e `SERVICE_ROLE_KEY` **têm que ser assinadas com o `JWT_SECRET` desta
instalação**. Trocar o segredo e não regerar as duas derruba auth, storage e realtime.

## 36 não-segredos — valor sugerido

### Banco e conexão
| variável | sugerido | nota |
|---|---|---|
| `POSTGRES_HOST` | `db` | nome do serviço |
| `POSTGRES_PORT` | `5432` | |
| `POSTGRES_DB` | `postgres` | o banco da aplicação entra como schema `app` aqui |

### Domínio
| variável | sugerido |
|---|---|
| `API_EXTERNAL_URL` | `https://$(PRIMARY_DOMAIN)` |
| `SUPABASE_PUBLIC_URL` | `https://$(PRIMARY_DOMAIN)` |
| `SITE_URL` | `https://$(PRIMARY_DOMAIN)` |
| `ADDITIONAL_REDIRECT_URLS` | *(vazio)* |

### Studio
| variável | sugerido |
|---|---|
| `DASHBOARD_USERNAME` | `pxz` (não `supabase`) |
| `STUDIO_DEFAULT_ORGANIZATION` | `MedLiv` |
| `STUDIO_DEFAULT_PROJECT` | `prontuario-xz` |

### Auth — o conjunto fechado, e é de propósito
| variável | sugerido | **difere do modelo** |
|---|---|---|
| `DISABLE_SIGNUP` | `true` | **sim** (modelo: `false`) — não há auto-cadastro; quem cria pessoa é `app.user_create` |
| `ENABLE_EMAIL_SIGNUP` | `true` | não — irrelevante com signup desligado |
| `ENABLE_EMAIL_AUTOCONFIRM` | `false` | não |
| `ENABLE_PHONE_SIGNUP` | `false` | **sim** (modelo: `true`) — não há provedor de SMS |
| `ENABLE_PHONE_AUTOCONFIRM` | `false` | **sim** (modelo: `true`) |
| `ENABLE_ANONYMOUS_USERS` | `false` | não |
| `JWT_EXPIRY` | `3600` | não |

### SMTP — vazio por enquanto
| variável | sugerido |
|---|---|
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_SENDER_NAME` | *(vazio)* |
| `SMTP_ADMIN_EMAIL` | um endereço real da equipe |
| `MAILER_URLPATHS_CONFIRMATION` / `_INVITE` / `_RECOVERY` / `_EMAIL_CHANGE` | `/auth/v1/verify` (as quatro) |

Sem SMTP o GoTrue não envia e-mail: convite, confirmação e recuperação não
funcionam. Irrelevante até a Etapa 4, que é quando o Auth entra.
**Não medido:** não subi o GoTrue para confirmar que ele tolera SMTP vazio.

### Storage
| variável | sugerido | nota |
|---|---|---|
| `GLOBAL_S3_BUCKET` | `stub` | backend é `file`; aqui é nome de diretório |
| `STORAGE_TENANT_ID` | `stub` | |
| `REGION` | `stub` | |
| `IMGPROXY_AUTO_WEBP` | `true` | |

### Supavisor (pooler)
| variável | sugerido |
|---|---|
| `POOLER_TENANT_ID` | `pxz` |
| `POOLER_DB_POOL_SIZE` | `5` |
| `POOLER_DEFAULT_POOL_SIZE` | `20` |
| `POOLER_MAX_CLIENT_CONN` | `100` |

Se a aplicação um dia conectar **pelo supavisor em modo transação**, vale a regra
já escrita no CLAUDE.md: `SET LOCAL` dentro de transação, nunca `SET` puro — com
pooler de transação a conexão é reciclada e o valor vaza para o próximo usuário.
O projeto já faz isso em `withTenant`.

### Studio / PostgREST
| variável | sugerido | nota |
|---|---|---|
| `PGRST_DB_SCHEMAS` | `public,storage,graphql_public` | continua obrigatória sem o `rest`: o Studio a lê |

## Com default no compose — só se quiser mudar
`PGRST_DB_MAX_ROWS` (1000), `PGRST_DB_EXTRA_SEARCH_PATH` (public), `OPENAI_API_KEY` (vazio),
`ANON_KEY_ASYMMETRIC`, `SERVICE_ROLE_KEY_ASYMMETRIC`, `SUPABASE_PUBLISHABLE_KEY`,
`SUPABASE_SECRET_KEY` (as quatro vazias — são as chaves opacas novas, não usadas aqui).

## Sobras do que saiu
`LOGFLARE_PUBLIC_ACCESS_TOKEN`, `LOGFLARE_PRIVATE_ACCESS_TOKEN` e
`DOCKER_SOCKET_LOCATION` não são mais lidas por ninguém. Não precisam existir.
