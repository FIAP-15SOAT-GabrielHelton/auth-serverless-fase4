# auth-serverless-fase4

**Porta de entrada serverless do OS Service**: o API Gateway e as Lambdas de autenticação de clientes via CPF e autorização (RBAC) do projeto **Oficina Mecânica** — Fase 4 do Tech Challenge FIAP.

Mantido da Fase 3, agora na frente do [`os-service`](https://github.com/FIAP-15SOAT-GabrielHelton/os-service), que é quem emite os tokens. O `billing-service` e o `execution-service` validam o mesmo JWT (segredo compartilhado) e são acessados pelos seus próprios endereços.

Na AWS, API Gateway e Lambda formam um par indissociável (é o Gateway que torna a Lambda invocável via HTTP), por isso o provisionamento do API Gateway vive aqui, e não no `k8s-infra-fase4`: ele só pode ser criado *depois* de todo o resto (Lambdas, endereço público do OS Service) já existir, então não caberia num repositório que é o *primeiro* a ser implantado. Ver ADR 5 e ADR 7 da RFC-001.

Decisões de autenticação (ADRs) na [RFC-001](https://github.com/FIAP-15SOAT-GabrielHelton/os-service/blob/main/docs/fase3/RFC-001-authentication-authorization-serverless.md), da Fase 3. A visão geral da Fase 4 — arquitetura, Saga e os sete repositórios — está no README do [`os-service`](https://github.com/FIAP-15SOAT-GabrielHelton/os-service#arquitetura) e na [RFC-005](https://github.com/FIAP-15SOAT-GabrielHelton/os-service/blob/main/docs/fase4/RFC-005-microsservicos-e-saga.md).

| Repositório | Responsabilidade |
| :--- | :--- |
| [`os-service`](https://github.com/FIAP-15SOAT-GabrielHelton/os-service) | Microsserviço de Ordens de Serviço (cadastros, estoque e ciclo de vida da OS) — PostgreSQL |
| [`billing-service`](https://github.com/FIAP-15SOAT-GabrielHelton/billing-service) | Microsserviço de orçamento e pagamento (Mercado Pago) — PostgreSQL |
| [`execution-service`](https://github.com/FIAP-15SOAT-GabrielHelton/execution-service) | Microsserviço de fila e andamento da execução — MongoDB |
| [`k8s-infra-fase4`](https://github.com/FIAP-15SOAT-GabrielHelton/k8s-infra-fase4) | VPC, EKS, RabbitMQ, MongoDB e New Relic |
| [`db-infra-fase4`](https://github.com/FIAP-15SOAT-GabrielHelton/db-infra-fase4) | RDS PostgreSQL, com usuário e bancos isolados por serviço |
| `auth-serverless-fase4` (este repo) | API Gateway e Lambdas de autenticação/RBAC |
| [`deploy-orchestrator-fase4`](https://github.com/FIAP-15SOAT-GabrielHelton/deploy-orchestrator-fase4) | Deploy e destroy de todos os repositórios, em sequência |

## Tecnologias utilizadas

| Categoria | Tecnologia |
| :--- | :--- |
| Linguagem | TypeScript (Node.js 20.x) |
| Runtime serverless | AWS Lambda (Node.js 20.x), AWS API Gateway (HTTP API v2) |
| Bundling | esbuild (um bundle CJS por Lambda, sem `node_modules` no zip) |
| Autenticação | `jsonwebtoken` (HS256 — mesmo segredo do `Auth::JwtEncoder` do OS Service) |
| Dev local | Express (`local_server.ts`), Docker Compose |
| Testes | Jest + ts-jest |
| IaC | Terraform (`hashicorp/aws` ~> 5.0) |
| CI/CD | GitHub Actions (`ci.yml` em toda PR/push; `cd_deploy.yml`/`cd_destroy.yml` manuais) |

## Arquitetura

```mermaid
flowchart TB
    Client["Cliente / Staff"]

    subgraph ThisRepo["auth-serverless-fase4 — este repositório"]
        APIGW["API Gateway (HTTP API v2)\núnica porta de entrada pública"]
        AuthLambda["Lambda auth_customer\n(AWS_PROXY)"]
        AuthzLambda["Lambda lambda_authorizer\n(REQUEST)"]
    end

    Rails["OS Service\n(repo os-service, via ELB)"]

    Client -- "POST /auth/customer" --> APIGW
    APIGW -- AWS_PROXY --> AuthLambda
    AuthLambda -- "POST /api/v1/auth/customer" --> Rails

    Client -- "demais rotas (Bearer JWT)" --> APIGW
    APIGW -- valida token --> AuthzLambda
    APIGW -- "HTTP_PROXY (sem VPC Link)" --> Rails

    classDef repo fill:#9bb8ff,stroke:#5470c6,color:#000
    classDef ext fill:#dddddd,stroke:#999999,color:#333,stroke-dasharray: 3 3
    class APIGW,AuthLambda,AuthzLambda repo
    class Rails,Client ext
```

## O que tem aqui

- **`auth_customer` (Lambda, `AWS_PROXY`)** — `POST /api/v1/auth/customer`: valida o formato do CPF (Módulo 11) e delega ao OS Service (Single Source of Truth) a existência/status do cliente e a emissão do JWT. Rota pública, sem authorizer.
- **`lambda_authorizer` (Lambda, `REQUEST`)** — valida a assinatura (HS256) e expiração do JWT usando o mesmo segredo do `Auth::JwtEncoder` do OS Service, e injeta `{ userId, role, type, cpf }` no contexto repassado ao backend. Não decide RBAC por rota — isso é responsabilidade do OS Service (defesa em profundidade).
- **API Gateway (HTTP API v2)** — porta de entrada pública do OS Service. Rotas públicas (`/auth/login`, `/up`, `/tracking/{protocol}`, `/webhooks/{proxy+}`, `/auth/customer`) fazem `HTTP_PROXY` direto ao ELB público do OS Service (sem VPC Link — ver ADR 5 da RFC-001); a rota protegida `ANY /api/v1/{proxy+}` passa pelo `lambda_authorizer`.

## Documentação da API

Este repositório não mantém uma spec OpenAPI própria — ele é um proxy fino na frente do OS Service, cujo contrato (payloads, status codes, schemas) já está documentado no Swagger do repositório [`os-service`](https://github.com/FIAP-15SOAT-GabrielHelton/os-service#documentação-da-api). O único endpoint específico deste repositório (`POST /auth/customer`) está descrito na seção ["O que tem aqui"](#o-que-tem-aqui) acima e na [RFC-001 §5](https://github.com/FIAP-15SOAT-GabrielHelton/os-service/blob/main/docs/fase3/RFC-001-authentication-authorization-serverless.md).

## Desenvolvimento local

```bash
docker compose up --build
```

Sobe o servidor local (Express, porta `3001`) simulando o comportamento das rotas do API Gateway, apontando para umo OS Service rodando em `http://host.docker.internal:3000` (o `os-service` rodando localmente).

```bash
curl -X POST http://localhost:3001/auth/customer \
  -H "Content-Type: application/json" \
  -d '{"cpf": "52998224725"}'
```

Cenários esperados: CPF válido e cliente ativo → `200` com `access_token`; CPF com formato/dígitos inválidos → `422`; cliente inexistente ou inativo → `401` (repassado do OS Service).

Sem Docker:

```bash
npm install
npm run dev       # servidor local, porta 3001 (lê PORT/JWT_SECRET/RAILS_API_BASE_URL do ambiente)
npm test          # Jest
npm run typecheck # tsc --noEmit
npm run lint      # eslint
```

## CI

Workflow `CI` (`.github/workflows/ci.yml`), disparado em toda Pull Request e em push para `main` — status check exigido pela proteção da branch antes do merge. Passos do job `test`:

1. **Install dependencies** (`npm ci`).
2. **Type check** (`npm run typecheck`) — TypeScript sem `any` implícito escondendo erro de tipo nos handlers/clients.
3. **Lint** (`npm run lint`).
4. **Run tests** (`npm test -- --ci`) — Jest cobrindo `auth_customer`, `rails_client` e o validador de CPF.

## Deploy

Workflow `CD Deploy (Lambdas & API Gateway)` (`.github/workflows/cd_deploy.yml`, `workflow_dispatch`), disparado manualmente com as credenciais temporárias da sessão do AWS Academy. Passos do job `deploy`:

1. **Mask Sensitive Credentials** — mascara credenciais AWS, `JWT_SECRET` e `NEW_RELIC_LICENSE_KEY` no log do Actions.
2. **Install dependencies** + **Build Lambda Bundles** (`npm run build`) — empacota cada handler via esbuild em `build/*.zip`, formato que o Terraform espera para o `aws_lambda_function`.
3. **Configure AWS Credentials** — autentica a sessão via `aws-actions/configure-aws-credentials`.
4. **Bootstrap S3 Backend** — reutiliza (ou cria) o bucket S3 compartilhado de state do Terraform entre os repositórios.
5. **Terraform Provisioning** — `terraform init` + `terraform apply -auto-approve`, provisionando as duas Lambdas, o API Gateway (rotas públicas/protegidas e o authorizer JWT) e lendo a URL pública do OS Service do SSM (publicada pelo repositório `os-service`). Ao final, imprime a URL do API Gateway como output do job.

**Pré-requisito:** o repositório `os-service` precisa ter sido implantado antes (publica `/oficina-mecanica/rails_api_base_url` no SSM Parameter Store, que este repositório lê).

**Ordem de deploy do projeto**: `k8s-infra-fase4` → `db-infra-fase4` → `os-service` → `billing-service` → `execution-service` → `auth-serverless-fase4` (este repo, por último), ou use o [`deploy-orchestrator`](https://github.com/FIAP-15SOAT-GabrielHelton/deploy-orchestrator-fase4).

### Configuração necessária

Secrets do repositório:
- `JWT_SECRET` — **precisa ser exatamente o mesmo valor** usado pelos três microsserviços (secret `JWT_SECRET` em `os-service`, `billing-service` e `execution-service`), já que o `lambda_authorizer` verifica a assinatura HS256 dos tokens emitidos pelo `Auth::JwtEncoder` do Rails.
- `NEW_RELIC_LICENSE_KEY`, `NEW_RELIC_ACCOUNT_ID` — usados pela extensão New Relic nas duas Lambdas (ver [Observabilidade](#observabilidade)).

## Observabilidade

As duas Lambdas (`auth_customer`, `lambda_authorizer`) rodam com a [Lambda Layer do New Relic](https://layers.newrelic-external.com) (Node.js) — instrumentação automática de invocações, sem alterar o código do handler. Ambas emitem logs estruturados em JSON, incluindo o `requestId` do API Gateway. Nas rotas `HTTP_PROXY` (sem Lambda), o próprio API Gateway injeta o header `X-Request-Id` (mapeado de `$context.requestId` em `infra/apigateway.tf`) para correlacionar com os logs/traces do OS Service; na rota via Lambda (`auth_customer`), o mesmo id é propagado manualmente para o Rails via `postToRails`.

Detalhes de arquitetura: [`docs/fase3/architecture/component-diagram.md`](https://github.com/FIAP-15SOAT-GabrielHelton/os-service/blob/main/docs/fase3/architecture/component-diagram.md#4-monitoramento) (repositório `os-service`).

## Destroy

Workflow `CD Destroy (Lambdas & API Gateway)`. Deve rodar **antes** do destroy dos microsserviços.

## Limitações conhecidas

- O Gateway cobre só o OS Service. Billing e Execution são acessados pelos seus próprios endereços públicos e validam o JWT por conta própria; colocá-los atrás do Gateway é o próximo passo.
- A rota pública `/webhooks/{proxy+}` é herança da Fase 3: na Fase 4 os webhooks (aprovação de orçamento e Mercado Pago) passaram para o `billing-service`.
