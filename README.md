# infra-gtw-kong

[![Kong](https://img.shields.io/badge/Kong-OSS-003459.svg)](https://konghq.com/)
[![Kubernetes](https://img.shields.io/badge/GKE-Ingress%20Controller-326CE5.svg)](https://kubernetes.io/)

Gateway de borda do sistema Zera: Kong (OSS, modo Postgres) rodando como Ingress Controller no GKE.
É por ele que passam o app mobile, o painel web e os serviços entre si — autenticação, rotas por
serviço, rate limit e proteção de payload ficam aqui, não em cada microsserviço. Este repositório
guarda só infraestrutura como código (values do Helm e manifests); não há aplicação para compilar.
Os serviços atrás dele são o [`ms-administrative-core`](https://github.com/zera-interdisciplinar/ms-administrative-core),
o [`ms-inventory`](https://github.com/zera-interdisciplinar/ms-inventory), o
[`ms-analytics-bff`](https://github.com/zera-interdisciplinar/ms-analytics-bff), o
`ms-artificial-intelligence-core`, o preditivo de quebra, o
[`scrapy`](https://github.com/zera-interdisciplinar/scrapy) e o
[`zera-web-app`](https://github.com/zera-interdisciplinar/zera-web-app).

## Sumário

- [Domínio](#domínio)
- [Arquitetura](#arquitetura)
- [Stack](#stack)
- [Aplicando mudanças](#aplicando-mudanças)
- [Configuração](#configuração)
- [Rotas](#rotas)
- [Autenticação](#autenticação)
- [Proteção de borda](#proteção-de-borda)
- [Rotas do painel web](#rotas-do-painel-web)
- [Rollout de mudança breaking de autenticação](#rollout-de-mudança-breaking-de-autenticação)
- [Deploy](#deploy)

## Domínio

QA e produção rodam na **mesma instância Kong**; o que separa os ambientes é o namespace do
Kubernetes (`qa` e `production`) e o prefixo da rota. Em QA toda rota ganha o prefixo `/qa/`
(`/qa/inventory`); em produção ela vem sem prefixo (`/inventory`). Todas usam `strip-path`: o
serviço recebe o caminho sem o prefixo do gateway.

Há dois tipos de credencial, e é importante não confundi-los:

- **API key** (`apikey`, plugin `key-auth`) identifica o **app cliente**: um *consumer* por
  quem chama (`mobile-app-*`, `inventory-*`, `predictor-*`, `admin-core-*`).
- **JWT** (RS256, plugin `jwt`) identifica o **usuário**: emitido pelo `ms-administrative-core` e
  validado aqui contra a chave pública dele.

## Arquitetura

```
manifests/
├── qa/                  Plugins, consumers e Ingresses do namespace qa
├── prod/                Os mesmos, para o namespace production
└── argocd/              Applications do ArgoCD (gtw-kong-qa / gtw-kong-prod) e o namespace argocd
helm/
├── values-autopilot.yaml   Kong (chart oficial) no GKE Autopilot: LoadBalancer externo, Postgres, admin em 127.0.0.1
└── values-argocd.yaml      ArgoCD, com recursos enxutos
.github/workflows/deploy-kong.yml   Aplica os manifests no cluster
```

Cada serviço tem um arquivo próprio em `manifests/{qa,prod}/<serviço>.yaml` com seus `Ingress` e
`KongPlugin`. Autenticação (`kong-auth.yaml`, `jwt-auth.yaml`) e proteção global
(`gateway-protection.yaml`) ficam em arquivos à parte.

## Stack

| | |
|---|---|
| Gateway | Kong OSS (chart `kong/kong`), Ingress Controller (KIC 3.x), banco Postgres |
| Plataforma | GKE (cluster `zera-cluster`, `southamerica-east1`) em modo Autopilot |
| GitOps | ArgoCD (`gtw-kong-qa` / `gtw-kong-prod`), com `prune` e `selfHeal` |
| CI/CD | GitHub Actions, autenticando no GCP por Workload Identity Federation |

## Aplicando mudanças

Não há build local. Para validar um manifest antes do PR:

```bash
kubectl apply --dry-run=server -f manifests/qa/
```

O `selfHeal` do ArgoCD reverte qualquer alteração manual no cluster, então toda mudança de rota
ou plugin passa por PR aqui — **teste em QA primeiro** (`manifests/qa/`) antes de espelhar em
`manifests/prod/`.

## Configuração

### Secret da chave pública JWT (criar manualmente, uma vez por ambiente)

O `KongConsumer administrative-core-issuer` referencia um Secret com a chave pública RSA do
`ms-administrative-core` (a mesma do Secret `ms-administrative-core-jwt` lá). O KIC 3.x identifica
o tipo da credencial pelo label `konghq.com/credential=jwt` (não pelo antigo campo `kongCredType`).

```sh
# jwt-public.pem = a chave pública gerada no ms-administrative-core

# QA
kubectl create secret generic administrative-core-jwt-qa -n qa \
  --from-literal=algorithm=RS256 \
  --from-literal=key=ms-administrative-core-qa \
  --from-file=rsa_public_key=jwt-public.pem
kubectl label secret administrative-core-jwt-qa -n qa konghq.com/credential=jwt

# Produção
kubectl create secret generic administrative-core-jwt-prod -n production \
  --from-literal=algorithm=RS256 \
  --from-literal=key=ms-administrative-core-prod \
  --from-file=rsa_public_key=jwt-public.pem
kubectl label secret administrative-core-jwt-prod -n production konghq.com/credential=jwt
```

`key` **precisa** ser igual ao claim `iss` do token (`zera.jwt.issuer` no ms-administrative-core,
hoje `ms-administrative-core-qa` / `ms-administrative-core-prod`). Como a credencial JWT do Kong
exige `key` único no gateway inteiro (não por namespace), `key` leva o sufixo do ambiente — usar o
mesmo valor nos dois quebra o apply com `unique key constraint violated for key`.

Rotação da chave: atualize este Secret e o `ms-administrative-core-jwt` no mesmo momento, depois
`kubectl rollout restart` do ms-administrative-core.

### Outros Secrets

As credenciais de API key dos consumers (`mobile-app-key-auth-secret`, `inventory-key-auth-secret`,
`predictor-key-auth-secret`, `admin-core-key-auth-secret`) e o Secret TLS `gateway-tls` também
vivem no cluster, fora do Git, em cada namespace. O Postgres do próprio Kong usa o Secret
`kong-postgres-secret` (`pg_password`).

## Rotas

Prefixos de QA e de produção, respectivamente (`/qa/…` e sem prefixo). Todas as rotas abaixo
vivem em `www.zeratech.online`.

| Serviço | Rota | Plugins | Observação |
|---|---|---|---|
| `ms-administrative-core` | `/administrative` | `jwt-auth`, `api-key-auth` | API autenticada |
| `ms-administrative-core` | `/administrative/api/v1/auth/*`, `/administrative/api/v1/invitations/redeem`, `/administrative/.well-known/jwks.json` | `api-key-auth`, `auth-rate-limit` | Rotas públicas de login/cadastro; regex com prioridade 1000 |
| `ms-inventory` | `/inventory` | `jwt-auth`, `api-key-auth` | API autenticada |
| `ms-inventory` | `/inventory/mcp` | `api-key-auth`, `ms-inventory-path-rewrite` | Ferramentas MCP para clientes de IA, só com API key |
| `ms-analytics-bff` | `/analytics` | `jwt-auth`, `api-key-auth` | SSE do dashboard |
| `ms-artificial-intelligence-core` | `/ai-core` | `api-key-auth` | |
| `ml-failure-predictor` | `/predictor` (MCP), `/predictor-rest` | `api-key-auth` (+ `mcp-path-rewrite` no MCP) | Serviço na porta 8000 |
| `scrapy` | `/scrapy/v1/boot`, `/scrapy/v1/flags` | rewrite de URI, sem plugin do Kong | O scrapy valida a `apikey` contra as keys do próprio banco (ver abaixo) |
| `scrapy` | `/scrapy` | nenhum | Painel admin |
| `scrapy` (só QA) | `/qa/scrapy/v1/bootstrap`, `/qa/scrapy/v1/connect` | `api-key-auth` + rewrite de URI | O scrapy atual não expõe essas rotas; resquício de uma versão anterior |
| `ms-administrative-core`, `ms-inventory` (só QA) | `/qa/<serviço>/swagger-ui`, `/qa/<serviço>/api-docs` | nenhum | Documentação pública em QA, de propósito |
| `zera-web-app` | hosts próprios | ver [Rotas do painel web](#rotas-do-painel-web) | |

**Por que o scrapy não usa `api-key-auth`:** o plugin só aceita as keys cadastradas como Secret no
Kong, e as keys do app mobile são geradas no painel do scrapy. Além disso, `strip-path` remove o
caminho **inteiro** configurado na regra, não só o prefixo — por isso cada subcaminho do scrapy tem
seu próprio `Ingress` e um `request-transformer` que reescreve a URI para exatamente o que o backend
espera (`/v1/boot`, `/v1/flags`). Sem isso o backend recebia `/`, caía no fallback que serve a UI e
o app mobile recebia `200` com HTML no lugar de JSON.

## Autenticação

| Camada | Onde | O quê |
|---|---|---|
| `api-key-auth` (`key-auth`) | Rotas de API e de `/auth/*`, JWKS, MCP | Identifica o app cliente (header `apikey`); consumers `mobile-app-*`, `inventory-*`, `predictor-*`, `admin-core-*` |
| `jwt-auth` (`jwt`) | Rotas de API de `ms-administrative-core`, `ms-inventory` e `ms-analytics-bff` | Valida o access token RS256 do usuário; casa o claim `iss` com a credencial do consumer `administrative-core-issuer` |
| `auth-rate-limit` (`rate-limiting`) | Rotas públicas `/auth/*` | 20 req/min por IP no login |

As rotas de API carregam os dois plugins, então o cliente envia **as duas** credenciais: `apikey`
(app) e `Authorization: Bearer <token>` (usuário) — é o que o app Android e o painel web já fazem.
Já `/api/v1/auth/login` (e `refresh`/`logout`) exigem **apenas** a API key: o cliente autentica com
a key, recebe o JWT e passa a mandar `Authorization: Bearer` nas demais chamadas.

O Ingress `ms-administrative-core-public-*` usa uma rota **regex com lookahead** para remover só o
prefixo e deixar `/api/v1/auth/...` chegar ao backend — valide com uma chamada real de login antes
de aplicar em produção.

## Proteção de borda

Não é firewall de VPC. O LoadBalancer do Kong é **público em 80/443** e o admin do Kong escuta só em
`127.0.0.1`. Não dá para fechar a origem por IP: ChatGPT/Claude saem de ranges que mudam.

Já existia: `apikey` / JWT nas APIs e `auth-rate-limit` (20/min) no login. Além disso,
`manifests/qa/gateway-protection.yaml` aplica no **gateway inteiro** (`KongClusterPlugin` global):

| Plugin | Teto |
|---|---|
| `gateway-rate-limit` | 120 req/min e 3000/h por IP (`policy: local`) |
| `gateway-request-size-limit` | Body até 4 MB |

Isso corta flood e payload gigante. Não substitui Cloud Armor (WAF L7) — o LB atual é do tipo
Network/External; o Armor entra se um dia o proxy for um HTTP(S) Load Balancer.

## Rotas do painel web

| Ambiente | Host | Namespace | Backend |
|---|---|---|---|
| QA | `qa.zeratech.online` | `qa` | `zera-web-app:80` |
| Produção | `gestao.zeratech.online` | `production` | `zera-web-app:80` |

Os Ingresses usam host explícito e encaminham o caminho completo. Assim, `/login`, as rotas da SPA
e `/api/*` chegam ao servidor do web, que serve os arquivos e executa o proxy com allowlist na mesma
origem. A chave do gateway fica no Secret do workload e o JWT continua sendo validado nas APIs;
**não aplique `jwt-auth`/`api-key-auth` ao HTML de login**. O buffering de resposta fica
desativado para preservar o SSE do dashboard. Timeouts de leitura/escrita pertencem ao Service do
web, mantido no repositório `zera-web-app`.

A raiz de `www.zeratech.online` e as rotas existentes das APIs permanecem com seus backends. Os
novos hosts não criam uma rota genérica para outros domínios.

Antes de disponibilizar cada host, confira o DNS para o LoadBalancer do Kong, o Secret TLS
`gateway-tls` no namespace correspondente (o certificado deve cobrir o host) e o Service com
endpoints saudáveis. A origem exige HTTPS e redireciona HTTP com 308; o proxy do Cloudflare deve
usar HTTPS na origem, com validação do certificado. Esta configuração segue a convenção TLS do
repositório e não provisiona certificados nem altera DNS.

O web de QA já possui workload. Em produção, o Ingress fica preparado para a primeira Release do
app: criar a rota não inicia Deployments nem bancos.

## Rollout de mudança breaking de autenticação

Roteiro seguido para exigir JWT nas rotas de API, útil para qualquer mudança que quebre o contrato
de autenticação do app:

1. `ms-administrative-core` no ar com `/auth/*` e JWKS.
2. Criar os Secrets `administrative-core-jwt-{qa,prod}` ([Configuração](#configuração)).
3. App mobile atualizado: faz `POST /auth/login` e passa `Bearer` nas demais chamadas.
4. Merge da mudança nos manifests → Kong passa a exigir o JWT nas rotas de API.
5. `ms-inventory` com resource server antes ou junto do passo 4.

## Deploy

O workflow `deploy-kong.yml` roda em todo push na `main` que mexa em `manifests/**` ou `helm/**`:
autentica no GCP por Workload Identity Federation, aplica `manifests/qa/` e `manifests/prod/` com
`kubectl apply` e, quando `helm/values-autopilot.yaml` muda, faz `helm upgrade` do Kong. Em paralelo,
o ArgoCD sincroniza `manifests/qa` e `manifests/prod` (Applications `gtw-kong-qa` e `gtw-kong-prod`,
com `prune` e `selfHeal`) — por isso uma mudança manual no cluster é desfeita. Reverta alterações
do Kong sempre no Git.
