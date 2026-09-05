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

```sh
# jwt-public.pem = a chave pública gerada no ms-administrative-core

# QA
kubectl create secret generic administrative-core-jwt-qa -n qa \
  --from-literal=kongCredType=jwt \
  --from-literal=algorithm=RS256 \
  --from-literal=key=ms-administrative-core \
  --from-file=rsa_public_key=jwt-public.pem

# Produção
kubectl create secret generic administrative-core-jwt-prod -n production \
  --from-literal=kongCredType=jwt \
  --from-literal=algorithm=RS256 \
  --from-literal=key=ms-administrative-core \
  --from-file=rsa_public_key=jwt-public.pem
```

`key` **precisa** ser igual ao claim `iss` do token (`zera.jwt.issuer` no
ms-administrative-core, hoje `ms-administrative-core`).

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
