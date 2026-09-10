# Ambiente — as 27 variáveis

Vão em **Ambiente**, no serviço `supabase` do EasyPanel. Nenhuma neste repositório.
O EasyPanel interpola `$(PRIMARY_DOMAIN)` no próprio valor.

**Eram 46 até 2026-09-10.** A decisão de usar o Supabase **só como autenticação**
tirou seis serviços (`studio`, `meta`, `storage`, `imgproxy`, `realtime`,
`supavisor`) e, com eles, 19 variáveis. A lista abaixo foi **medida** contra o
`docker-compose.yml`, não escrita de memória: são as que sobraram sem default.

## 4 SEGREDOS — gerar, nunca reaproveitar de exemplo

`utils/generate-keys.sh` gera o conjunto de JWT de uma vez (ele ainda imprime as
antigas; ignore as que não estão aqui).

| variável | como gerar |
|---|---|
| `POSTGRES_PASSWORD` | `openssl rand -base64 36` (sem `/` e `+` se o painel implicar) |
| `JWT_SECRET` | `openssl rand -hex 32` — mínimo 32 caracteres |
| `ANON_KEY` | JWT `role: anon` assinado com o `JWT_SECRET` — `utils/generate-keys.sh` |
| `SERVICE_ROLE_KEY` | JWT `role: service_role` com o mesmo segredo — idem |

`ANON_KEY` e `SERVICE_ROLE_KEY` **têm que ser assinadas com o `JWT_SECRET` desta
instalação**, e agora isso é mais do que uma regra de configuração: o Kong aceita
a chave como `apikey`, mas **o GoTrue valida a ASSINATURA do bearer**. Medido em
2026-09-10: uma chave que não é JWT passa pelo Kong e o GoTrue responde
`bad_jwt`. As duas coisas são conferidas em lugares diferentes.

**`SMTP_PASS` também é segredo**, quando existir. Ela está entre as não-secretas
abaixo porque hoje é vazia — ver o bloqueio.

## 23 não-segredos — valor sugerido

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

`SUPABASE_PUBLIC_URL` **saiu**: era do Studio e do Storage.

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
`OPENAI_API_KEY` (vazio), `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SECRET_KEY`,
`ANON_KEY_ASYMMETRIC`, `SERVICE_ROLE_KEY_ASYMMETRIC` (as quatro vazias — são as
chaves opacas novas, não usadas aqui).

## O que SAIU em 2026-09-10, com os seis serviços
`SECRET_KEY_BASE`, `VAULT_ENC_KEY`, `PG_META_CRYPTO_KEY`, `DASHBOARD_USERNAME`,
`DASHBOARD_PASSWORD`, `S3_PROTOCOL_ACCESS_KEY_ID`,
`S3_PROTOCOL_ACCESS_KEY_SECRET`, `SUPABASE_PUBLIC_URL`, `STUDIO_DEFAULT_ORGANIZATION`,
`STUDIO_DEFAULT_PROJECT`, `GLOBAL_S3_BUCKET`, `STORAGE_TENANT_ID`, `REGION`,
`IMGPROXY_AUTO_WEBP`, `POOLER_TENANT_ID`, `POOLER_DB_POOL_SIZE`,
`POOLER_DEFAULT_POOL_SIZE`, `POOLER_MAX_CLIENT_CONN`, `PGRST_DB_SCHEMAS`.

Podem ser apagadas do Ambiente. **As duas de `DASHBOARD_` são as do Studio**, e
apagá-las é o registro de que aquela porta não existe mais.
