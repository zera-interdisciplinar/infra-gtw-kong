# infra-gtw-kong

Kong (OSS, modo Postgres) rodando como Ingress Controller no GKE. Os manifests em
`manifests/qa` e `manifests/prod` são aplicados pelo workflow `deploy-kong` e
sincronizados pelo ArgoCD (`gtw-kong-qa` / `gtw-kong-prod`).

## Autenticação

| Camada | Onde | O quê |
|--------|------|-------|
| `api-key-auth` (`key-auth`) | rotas `/auth/*`, JWKS e swagger | identifica o app cliente (header `apikey`), consumer `mobile-app-*` |
| `jwt-auth` (`jwt`) | rotas de API de `ms-administrative-core` e `ms-inventory` | valida o access token RS256 do usuário; casa o claim `iss` com a credencial do consumer `administrative-core-issuer` |
| `auth-rate-limit` (`rate-limiting`) | rotas `/auth/*` | 20 req/min por IP no login |

Rotas de API exigem **apenas** o JWT. `/api/v1/auth/login` (e `refresh`/`logout`)
exigem **apenas** a API key — o cliente autentica com a key, recebe o JWT e passa a
mandar `Authorization: Bearer <token>` nas demais chamadas.

### Secret da chave pública JWT (criar manualmente, uma vez por ambiente)

O `KongConsumer administrative-core-issuer` referencia um Secret com a chave pública
RSA do `ms-administrative-core` (a mesma do Secret `ms-administrative-core-jwt` lá).

O KIC 3.x identifica o tipo da credencial pelo label `konghq.com/credential=jwt`
(não pelo antigo campo `kongCredType`).

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

`key` **precisa** ser igual ao claim `iss` do token (`zera.jwt.issuer` no
ms-administrative-core, hoje `ms-administrative-core-qa`/`ms-administrative-core-prod`).

QA e produção rodam na **mesma instância Kong**, e a credencial JWT do Kong exige
`key` único no gateway inteiro (não por namespace) — por isso `key` leva o sufixo
do ambiente. Usar o mesmo valor nos dois quebra o apply com
`unique key constraint violated for key`.

Rotação da chave: atualize este Secret e o `ms-administrative-core-jwt` no mesmo
momento, depois `kubectl rollout restart` do ms-administrative-core.

## Rollout (mudança breaking para o app mobile)

1. `ms-administrative-core` no ar com `/auth/*` e JWKS (feito no repo do serviço).
2. Criar os Secrets `administrative-core-jwt-{qa,prod}` (acima).
3. App mobile atualizado: faz `POST /auth/login` e passa `Bearer` nas demais chamadas.
4. Merge deste PR → Kong passa a exigir o JWT nas rotas de API.
5. `ms-inventory` com resource server (PR próprio) antes ou junto do passo 4.

Testar em QA primeiro: o Ingress `ms-administrative-core-public-*` usa uma rota
**regex com lookahead** para remover só o prefixo e deixar `/api/v1/auth/...` chegar
ao backend — validar com uma chamada real de login antes de aplicar em produção.

## Proteção de borda (não é firewall de VPC)

O LoadBalancer do Kong é **público em 80/443**. O admin do Kong escuta só em `127.0.0.1`.
Não dá para fechar a origem por IP: ChatGPT/Claude saem de ranges que mudam.

Já existia: `apikey` / JWT nas APIs e `auth-rate-limit` (20/min) no login.

`manifests/qa/gateway-protection.yaml` aplica no **gateway inteiro** (`KongClusterPlugin`):

| Plugin | Teto |
| --- | --- |
| `gateway-rate-limit` | 120 req/min e 3000/h por IP (`policy: local`) |
| `gateway-request-size-limit` | body até 4 MB |

Isso corta flood e payload gigante. Não substitui Cloud Armor (WAF L7) — o LB atual é do tipo Network/External, Armor entra se um dia o proxy for um HTTP(S) Load Balancer.


## Rotas do painel web

| Ambiente | Host | Namespace | Backend |
|---|---|---|---|
| QA | `qa.zeratech.online` | `qa` | `zera-web-app:80` |
| Producao | `gestao.zeratech.online` | `production` | `zera-web-app:80` |

Os Ingresses usam host explicito e encaminham o caminho completo. Assim, `/login`, as rotas da SPA e `/api/*` chegam ao servidor do web, que serve os arquivos e executa o proxy allowlist na mesma origem. A chave do gateway fica no Secret do workload e o JWT continua sendo validado nas APIs; nao aplicar `jwt-auth`/`api-key-auth` ao HTML de login. O buffering de resposta fica desativado para preservar o SSE do dashboard. Timeouts de leitura/escrita pertencem ao Service do web, mantido no repositorio `zera-web-app`.

A raiz de `www.zeratech.online` e as rotas existentes das APIs permanecem com seus backends. Os novos hosts nao criam uma rota generica para outros dominios.

Antes de disponibilizar cada host, conferir o DNS para o LoadBalancer do Kong, o Secret TLS `gateway-tls` no namespace correspondente (certificado deve cobrir o host) e o Service com endpoints saudaveis. A origem exige HTTPS e redireciona HTTP com 308; o proxy do Cloudflare deve usar HTTPS na origem, com validacao do certificado. Esta mudanca segue a convencao TLS do repositorio e nao provisiona certificados nem altera DNS.

O web de QA ja possui workload. Em producao, o Ingress fica preparado para a primeira Release do app: criar a rota nao inicia Deployments nem bancos. O workflow existente e o ArgoCD aplicam os arquivos de `manifests/qa` e `manifests/prod` conforme o fluxo atual da infraestrutura; esta mudanca nao troca branches de GitOps nem os gatilhos de deploy dos aplicativos.
