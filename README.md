# platform-iam (Config) - Ecossistema Hubinity - Active
> Parte integrante do ecossistema distribuído Hubinity.

---

## 💻 Visão Geral
- **O que faz:** Source of truth declarativo da configuração de Identidade e Acesso (IAM) da Hubinity. Versiona, como JSON, os realms Keycloak (`hibit` e `star-coffee`) com todos os clients, roles, scopes, mappers e usuários de teste — prontos para serem importados em qualquer instância Keycloak 26.x (local ou cloud).
- **Problema que resolve:** Sem este repositório, cada ambiente (dev/staging/prod) precisaria recriar manualmente a configuração de autenticação, gerando divergência entre realms e quebra dos backends/frontends que dependem de `clientId`s, roles e scopes específicos. Aqui, qualquer mudança em IAM passa por PR e é replicada de forma idempotente.
- **Posicionamento no Ecossistema:** Camada de **configuração** consumida em runtime pelo Keycloak provisionado no `platform-infra` (dev) e Railway Hobby (cloud). Os JWTs emitidos pelos realms aqui descritos são validados pelos 4 backends Spring (catalog/support/cashier/order) e usados pelos 4 frontends Angular (catalog-web/support-web/cashier-web/totem-web).

## 🏗️ Papel na Arquitetura
- **Tipo de Componente:** Configuração declarativa (JSON) — **não contém código executável**.
- **Responsabilidades Principais:**
  - Definir o realm `hibit` (HiBit: loja/assistência técnica/caixa) com seus clients, roles e usuários DEV.
  - Definir o realm `star-coffee` (Star Coffee: cafeteria + totem PWA) com seus clients, roles e usuários DEV.
  - Garantir export 100% compatível com o schema partialImport/fullImport do Keycloak 26.x.
  - Servir como referência canônica para os `clientId`s e nomes de roles usados em `application.yaml` dos demais 11 repositórios.
- **Limites e Fronteiras (Boundaries):**
  - **Não** provisiona o servidor Keycloak (responsabilidade de `platform-infra` em dev e do checklist cloud em prod).
  - **Não** define federação cross-realm — a ponte `catalog:read` em `star-coffee` é apenas um scope declarado; a relação de confiança entre realms será wired no `platform-api-gateway` (Fase 5).
  - **Não** armazena segredos reais — os campos `secret` no JSON são placeholders dev-only.

## 🔗 Dependências e Comunicação
### Serviços Internos da Hubinity
- **Nenhuma dependência upstream.** Este repositório é folha — é consumido por:
  - `hb-catalog-service`, `hb-support-service`, `hb-cashier-service` (realm `hibit`, validação JWT como OAuth2 Resource Server)
  - `sc-order-service` (realm `star-coffee`, mesma validação)
  - `hb-catalog-web`, `hb-support-web`, `hb-cashier-web` (realm `hibit`, login PKCE)
  - `sc-totem-web` (realm `star-coffee`, login PKCE + cliente de device)
  - `platform-infra` (monta `realms/` como volume read-only no container Keycloak)

### Infraestrutura e Serviços Externos
- **Keycloak 26.0** (`quay.io/keycloak/keycloak:26.0`) — único runtime que importa estes JSONs no boot.

## 🛠️ Tecnologias e Ferramentas
| Camada | Tecnologia | Versão |
| :--- | :--- | :--- |
| Identity Provider | Keycloak | 26.0 |
| Formato dos arquivos | JSON (schema fullImport Keycloak) | — |
| Versionamento | Git + PR review | — |

## 📐 Padrões de Projeto e Arquitetura do Código
- **Estilo Arquitetural:** **Configuration as Code** — o estado desejado do IAM é descrito declarativamente em JSON e aplicado de forma idempotente.
- **Padrões Relevantes:**
  - **Single Source of Truth** — qualquer mudança feita via UI do Keycloak DEVE ser re-exportada para este repositório via `kc.sh export`.
  - **One realm per business unit** — `hibit` e `star-coffee` são realms isolados; a integração cross-tenant acontece no API Gateway.
  - **PKCE-only** para clients públicos (SPAs) via `pkce.code.challenge.method: S256`.
  - **Service Accounts** (client credentials grant) para comunicação backend-to-backend, com `defaultClientScopes` enxutos.

## 📂 Estrutura do Projeto
```text
platform-iam/
├── README.md
├── realms/
│   ├── hibit-realm.json             # realm `hibit` (full export — HiBit)
│   └── star-coffee-realm.json       # realm `star-coffee` (full export — Star Coffee)
└── clients/                         # reservado para overlays/patches por client (futuro)
```

Cada arquivo `*-realm.json` é um export completo que recria o realm do zero: settings, roles, clients, client scopes, users (com credenciais DEV) e protocol mappers — inline (`--users realm_file`) para deixar o diff legível em PR.

## ⚙️ Configuração e Variáveis de Ambiente
```bash
# Credenciais bootstrap do servidor Keycloak (provisionadas pelo runtime, NÃO por este repo)
KEYCLOAK_ADMIN=admin            # DEV ONLY
KEYCLOAK_ADMIN_PASSWORD=admin   # DEV ONLY

# Em produção (Railway), os segredos reais dos clients são injetados via env
# (ex.: KC_DB_PASSWORD, HB_CATALOG_CLIENT_SECRET) — os valores `secret` no JSON
# são SUBSTITUÍDOS no momento do import. Nunca commitar segredos reais aqui.
```

## 🚀 Como Instalar e Executar
### Pré-requisitos
- Docker (para rodar o Keycloak local)
- Acesso ao repositório `platform-infra` (que já orquestra o Keycloak via docker-compose)

### Passos para Instalação
Não há instalação — este repositório é puramente declarativo. Basta clonar:
```bash
git clone <repo-url> platform-iam
```

### Execução Local
A forma canônica é via `platform-infra`, que monta `./realms/` no container Keycloak:
```bash
# A partir do workspace root
cd platform-infra
make up                              # sobe postgres + rabbitmq + keycloak (default profile)
# Keycloak admin: http://localhost:8081  (admin / admin)
# Realms `hibit` e `star-coffee` já populados automaticamente
```

### Execução via Docker (standalone, sem o platform-infra)
```bash
# A partir da raiz deste repositório
docker run --rm \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  -v "$(pwd)/realms:/opt/keycloak/data/import:ro" \
  quay.io/keycloak/keycloak:26.0 \
  start-dev --import-realm
```

### Forçar re-import (sobrescrever realm existente)
```bash
docker exec -it <kc-container> \
  /opt/keycloak/bin/kc.sh import \
  --file /opt/keycloak/data/import/hibit-realm.json \
  --override true
```

### Exportar mudanças feitas via Admin UI
```bash
docker exec -it <kc-container> \
  /opt/keycloak/bin/kc.sh export \
  --dir /opt/keycloak/data/import \
  --realm hibit \
  --users realm_file
# copiar o arquivo para fora do container, diff com a versão do repo, abrir PR
```

## 🔄 Fluxos Principais

### Fluxo de mudança de IAM (workflow)
1. Edita-se o realm via Admin UI Keycloak local OU diretamente no JSON.
2. `kc.sh export` regenera o JSON canônico.
3. Diff + PR revisado por pelo menos uma pessoa.
4. Merge → próximo boot do Keycloak (dev) ou `kc.sh import --override true` (cloud) aplica a mudança.

### Usuários DEV (quick reference)

#### Realm `hibit` — http://localhost:8081/realms/hibit
| Username      | Senha       | Role             | Consumidor                       |
| ------------- | ----------- | ---------------- | -------------------------------- |
| `admin-hibit` | `admin123`  | `admin`          | qualquer web app HiBit           |
| `tech1`       | `tech123`   | `tecnico`        | `hb-support-web` (porta 4201)    |
| `atendente1`  | `atend123`  | `atendente`      | `hb-support-web` (porta 4201)    |
| `caixa1`      | `caixa123`  | `operador-caixa` | `hb-cashier-web` (porta 4202)    |

Backends (client credentials, sem login humano):
| `clientId`           | `secret` (DEV)        | Default scopes                  |
| -------------------- | --------------------- | ------------------------------- |
| `hb-catalog-service` | `dev-catalog-secret`  | `catalog:read`, `catalog:write` |
| `hb-support-service` | `dev-support-secret`  | `support:read`, `support:write` |
| `hb-cashier-service` | `dev-cashier-secret`  | `cashier:read`, `cashier:write` |

SPAs (PKCE, sem segredo):
| `clientId`       | Redirect URI              | Web origin               |
| ---------------- | ------------------------- | ------------------------ |
| `hb-catalog-web` | `http://localhost:4200/*` | `http://localhost:4200`  |
| `hb-support-web` | `http://localhost:4201/*` | `http://localhost:4201`  |
| `hb-cashier-web` | `http://localhost:4202/*` | `http://localhost:4202`  |

#### Realm `star-coffee` — http://localhost:8081/realms/star-coffee
| Username   | Senha      | Role                | Consumidor                  |
| ---------- | ---------- | ------------------- | --------------------------- |
| `admin-sc` | `admin123` | `admin`             | qualquer superfície Star    |
| `gerente1` | `ger123`   | `gerente-cafeteria` | gestão / fechamento totem   |

Clients (backend + dispositivo):
| `clientId`         | Tipo            | `secret` (DEV)             | Observações                                                                    |
| ------------------ | --------------- | -------------------------- | ------------------------------------------------------------------------------ |
| `sc-order-service` | confidential SA | `dev-order-secret`         | Carrega `catalog:read` (cross-realm trust planejado p/ gateway na Fase 5).     |
| `sc-totem-web`     | public PKCE     | —                          | PWA kiosk, porta 4203.                                                         |
| `device-totem`     | confidential    | `dev-device-totem-secret`  | Password grant + offline session ~30d para bootstrap do kiosk.                 |

### Cross-realm consideration: `catalog:read` em `star-coffee`
O serviço `sc-order-service` vive no realm `star-coffee` mas precisa consultar o catálogo HiBit. Modelamos isso por enquanto como **um scope `catalog:read` presente também no realm `star-coffee`**, de modo que os tokens emitidos lá já carreguem o claim. A federação real (token exchange ou JWT relayed) será implementada na camada do `platform-api-gateway` na **Fase 5** — este repositório NÃO configura identity federation entre realms hoje.

## 📊 Observabilidade e Testes
- **Logs & Tracing:** Nenhum mecanismo de tracing/observabilidade próprio nesta camada — observabilidade é do servidor Keycloak runtime (ver `platform-infra` profile `observability`).
- **Como Rodar os Testes:** Não há testes automatizados; validação se dá pela importação bem-sucedida no Keycloak 26.x — `make up` em `platform-infra` deve subir o Keycloak sem erros e ambos os realms ficam disponíveis em `/realms/hibit` e `/realms/star-coffee`.

---

## ⚠️ DEV ONLY — Aviso de Segurança
> **Todas as senhas e valores `secret` deste repositório existem APENAS para desenvolvimento local.**
> Nunca reutilizar em ambientes que tocam dados reais.
> Deploys em produção devem:
> - Substituir cada campo `secret` via injeção de env no momento do import, ou via Admin API logo após o primeiro import.
> - Substituir cada usuário de teste (`admin-hibit`, `tech1`, `atendente1`, `caixa1`, `admin-sc`, `gerente1`) por identidades reais — esses usuários DEV não devem sequer ser criados nos realms de produção.
> - Manter `bruteForceProtected: true` (já habilitado), aplicar uma password policy real (hoje usamos `length(6)` por conveniência DEV) e exigir email verification.

As variáveis `KC_DB_*` e as credenciais bootstrap do Keycloak admin são provisionadas por ambiente pelo `platform-infra/` e **não** fazem parte deste repositório.

## 📦 Compatibilidade de Schema
Estes arquivos miram o schema **partialImport / fullImport do Keycloak 26.x**. Escolhas notáveis:
- `protocol: openid-connect` em todos os clients; nenhum SAML.
- PKCE forçado via atributo per-client `pkce.code.challenge.method: S256`.
- `clientScopes` no nível do realm; opt-in por client via `defaultClientScopes`/`optionalClientScopes`.
- O client `device-totem` sobrescreve session attributes (`client.session.*`, `client.offline.session.*`) para atingir lifespan offline de ~30 dias, alinhado com o requisito do kiosk.

Se uma versão futura do Keycloak rejeitar algum desses campos, a correção fica isolada neste repo: re-export, substitui o JSON, abre PR.
