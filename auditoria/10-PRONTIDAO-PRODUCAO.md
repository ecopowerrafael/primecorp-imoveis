# 10 — Prontidão para produção

## O que existe

- Dockerfile não-root, healthcheck e Uvicorn (`Dockerfile:1-11`).
- Compose com DB, API, Redis e nginx (`docker-compose.yml:1-47`).
- `.env.example` exige segredo, senha, URL, cookie seguro e flags (`.env.example:1-19`).
- nginx redireciona HTTP/HTTPS, TLS e limites (`nginx/primecorp.conf:6-43`).
- Scripts de backup e restore de teste.

## Bloqueadores

### PROD-001 — CRITICA — publicação do painel não é segura

O README orienta não publicar `painel.html` no site (`README.md:155-159`), mas o painel é
o único frontend que aparenta executar os módulos internos e não está protegido por API.
Sem subdomínio/controle real, a operação não é publicável.

### PROD-002 — CRITICA — lead não chega a nenhum operador

`ENDPOINT` vazio em `web/index.html:758-765`; não existe rota de lead. Isso bloqueia a
função comercial principal.

### PROD-003 — ALTA — não há homologação/rollback/migration operacional

Compose inicializa DDL apenas em volume novo; não há pipeline, versão de schema ou rollback
documentado de forma executável. Documentação de deploy não comprova infraestrutura real.

### PROD-004 — ALTA — backup/restore e cron não comprovados

Scripts existem, mas nenhum agendamento, alerting ou execução foi verificado. O backup é
local por padrão (`scripts/backup.sh:4-10`).

### PROD-005 — ALTA — configurações de domínio/certificado são placeholders operacionais

nginx usa nomes/caminhos de certificados montados (`nginx/primecorp.conf:8,17-18`), mas
DNS, certificados, renovação e rede não são versionados/verificados.

### PROD-006 — MEDIA — observabilidade insuficiente

Erro genérico informa apenas que incidente foi registrado (`app/main.py:62-65`), mas não
há logger estruturado, correlation ID, métricas, tracing ou destino de alertas visível.

## Checklist de go-live

- [ ] integração real do painel e site
- [ ] tenant/ownership/RBAC aprovados
- [ ] migrations versionadas e ensaiadas
- [ ] secrets em secret manager/ambiente seguro
- [ ] TLS/DNS renovável e proxy headers restritos
- [ ] backup externo + restore observado
- [ ] rate limiting distribuído e monitorado
- [ ] testes E2E e autorização verdes em CI
- [ ] dados demonstrativos removidos
- [ ] revisão jurídica/LGPD e política de retenção
- [ ] alertas de erro, saúde, fila, banco e backup

## Achados adicionais

### PROD-007 — MEDIA — build/release não separa artefatos público e interno

**Evidência:** documentação diz que `web/` é artefato final, mas também contém
`painel.html` com PII/fixtures (`README.md:155-159`; `web/painel.html:381-417`).
**Impacto:** publicação acidental do painel e exposição de dados de demonstração.
**Recomendação:** pipelines/artefatos distintos, denylist de arquivos internos e smoke test
de conteúdo publicado.
**Confiança:** alta.
