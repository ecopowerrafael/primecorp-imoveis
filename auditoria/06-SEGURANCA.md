# 06 — Auditoria de segurança

## Controles positivos

- Boot fail-closed para segredos (`app/config.py:37-109`).
- Argon2id e comparação para usuário inexistente (`app/auth.py:183-186`).
- Sessão HttpOnly, Secure configurável e SameSite Strict (`app/auth.py:249-253`).
- MFA por papel e TOTP (`app/auth.py:209-230`).
- Headers de segurança/CSP (`app/main.py:44-59`).
- SQL parametrizado nas consultas observadas.
- nginx com TLS 1.2/1.3 e rate limit (`nginx/primecorp.conf:1-43`).
- Docker API como usuário não-root (`Dockerfile:3-8`).

## Achados

### SEC-001 — CRITICA — autenticação do painel é falsa

**Evidência:** `web/painel.html:381-388` preenche credenciais e `entrar()` não chama API,
apenas oculta/exibe elementos (`:1084-1088`).
**Exploração:** qualquer visitante que obtenha o painel acessa o console visual.
**Impacto:** controle de acesso inexistente no frontend e risco de decisões com dados falsos.
**Correção:** backend como autoridade, cookie real, logout revogável, tratamento 401 e
remoção de credenciais/PII hardcoded.
**Confiança:** alta.

### SEC-002 — ALTA — alteração arbitrária de contatos via opt-out

**Evidência:** `app/main.py:266-284`; sessão é o único requisito.
**Impacto:** integridade e disponibilidade de dados pessoais/comerciais.
**Correção:** autorização de titular/atendimento, escopo, confirmação e idempotência.
**Confiança:** alta.

### SEC-003 — ALTA — trust excessivo em headers de proxy

**Evidência:** `app/auth.py:55-60` usa primeiro `X-Forwarded-For`; `Dockerfile:11` inicia
Uvicorn com `--forwarded-allow-ips "*"`.
**Cenário:** se API ficar acessível diretamente ou houver proxy não confiável, cliente
forja IP e contorna rate limit/auditoria por IP.
**Correção:** restringir proxies confiáveis, não expor API diretamente e usar conexão
entre proxy/API com origem conhecida.
**Confiança:** alta para configuração; exploração depende da topologia real.

### SEC-004 — ALTA — dados de demonstração e PII no artefato de frontend

**Evidência:** nomes/CRECI em `web/painel.html:415-417`, `:1011-1027`, credenciais em
`:381-386`, arrays em `:1487-1503`.
**Impacto:** vazamento/publicação acidental e treinamento operacional com dados falsos.
**Correção:** separar fixture de desenvolvimento, remover do artefato de produção e revisar
  `.gitignore`/pipeline para impedir publicação do painel no domínio público.
**Confiança:** alta.

### SEC-005 — MEDIA — ausência de proteção explícita contra CSRF/origin

**Evidência:** autenticação usa cookie (`app/auth.py:249-253`), não há middleware de
  CSRF nem validação `Origin`/`Referer` em rotas mutáveis.
**Impacto:** SameSite Strict reduz o risco, mas não substitui defesa explícita em cenários
  same-site, subdomínios ou mudanças futuras de cookie.
**Correção:** token CSRF/origin allowlist para mutações e testes.
**Confiança:** média (controle ausente confirmado; exploração depende do deployment).

### SEC-006 — MEDIA — endpoint de saúde divulga versão do PostGIS e horário

**Evidência:** `app/main.py:72-75` é público e retorna `postgis_version()`.
**Impacto:** enumeração de stack; baixo isoladamente.
**Correção:** health público mínimo e endpoint de diagnóstico protegido.
**Confiança:** alta.

### SEC-007 — NÃO VERIFICADO — dependências, Docker e TLS em execução

Não foi feita instalação/scan de CVEs, nem teste de certificado, CORS, firewall, Redis,
PostgreSQL exposto ou rotação de segredos. Não há base para declarar ausência de falhas.
