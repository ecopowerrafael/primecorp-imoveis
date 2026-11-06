# 11 — Roadmap de correções

## P0 — bloquear operação enganosa e exposição

### R-01. Decisão de tenancy e autorização

**Dependências:** nenhuma. **Aceite:** arquitetura documenta single-tenant ou todas as
entidades têm tenant/ownership; testes de usuário A contra dado B falham com 403/404.

### R-02. Integrar autenticação do painel

**Dependências:** R-01. **Aceite:** login/MFA usam `/api/auth/login`, `/eu` hidrata usuário,
logout revoga sessão, 401 retorna à entrada, nenhum segredo/PII de fixture no artefato.

### R-03. Implementar captação pública

**Dependências:** R-01, contrato LGPD. **Aceite:** POST validado persiste uma vez, possui
base legal/source_ref/retention, rate limit e auditoria; erro de banco não mostra recibo.

### R-04. Corrigir opt-out e escopo de mandato

**Dependências:** R-01. **Aceite:** somente titular verificado/atendimento autorizado
  altera contato; criação de mandato exige papel e relações válidas; testes negativos cobrem
  IDs alheios e papel leitura.

## P1 — tornar o domínio transacional confiável

### R-05. API-first para painel

**Dependências:** R-02. **Aceite:** radar, anúncios, avaliação, fila, mandato e PTAM usam
  endpoints reais; cada ação possui loading, erro, retry seguro e ID persistido.

### R-06. Migrations e concorrência

**Dependências:** R-01. **Aceite:** migration versionada, dry-run/rollback documentados,
  sequence/unique para PTAM, idempotency key para import/lead e teste concorrente.

### R-07. Importação assíncrona

**Dependências:** R-06; decisão Redis/worker. **Aceite:** streaming, job status,
  retry/backoff, limite por usuário, cancelamento e histórico associado ao `run_id`.

### R-08. Retenção/DSR real

**Dependências:** R-03, R-04. **Aceite:** acesso/correção/anonimização/eliminação com
  autorização, trilha, retenção e prova de execução sem remover auditoria necessária.

## P2 — operação e escala

### R-09. Observabilidade

**Aceite:** logs estruturados sem segredos/PII excessiva, correlation ID, métricas de
  latência/erro/pool/fila e alertas de backup/restore.

### R-10. Imagens e uploads

**Aceite:** storage privado, MIME/tamanho, nomes não controlam path, hashes persistidos,
  processamento assíncrono e testes de decompression bomb/path traversal.

### R-11. Segurança de proxy e browser

**Aceite:** proxy allowlist, API não exposta diretamente, CSRF/origin para mutações,
  CSP no host estático e renderização sem `innerHTML` inseguro.

## P3 — acabamento e otimização

Calibrar mercado com fonte/data/responsável, melhorar acessibilidade, adicionar paginação
cursor e índices somente após `EXPLAIN`/métricas, evoluir HNSW/pgvector se volume justificar
e remover documentação histórica que não representa o produto atual.

## Estratégia de regressão

Cada item deve adicionar testes para caminho feliz, erro, autorização, tenant, duplicidade,
concorrência e dados nulos. O CI deve executar lint/syntax, pytest, testes JS em ambiente
isolado do diretório pai, integração com PostgreSQL efêmero, testes de contrato HTTP e
smoke de artefatos público/interno. Nenhum item é “pronto” apenas por alterar a UI.
