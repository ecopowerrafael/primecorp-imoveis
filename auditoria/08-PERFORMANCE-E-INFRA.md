# 08 — Performance e infraestrutura

## Configuração observada

Compose define PostGIS, API, Redis e proxy; DB usa WAL/archive e volumes (`docker-compose.yml:1-47`).
API tem pool 2–16 (`app/db.py:21-33`). nginx limita login a 5/min e API a 60/min
(`nginx/primecorp.conf:1-43`). Docker expõe healthcheck (`Dockerfile:10`).

## Riscos e achados

### PERF-001 — ALTA — tarefas pesadas síncronas no worker HTTP

**Evidência:** importação lê até 100 mil linhas e executa `executemany`/histórico dentro
  do request (`app/main.py:186-259`); Redis não tem worker.
**Impacto:** timeout, bloqueio de worker, consumo de memória e indisponibilidade.
**Recomendação:** job assíncrono com estado `queued/running/succeeded/failed`, retry,
  backoff, limite por usuário e idempotency key.
**Confiança:** alta.

### PERF-002 — MEDIA — captação faz múltiplas consultas e pode varrer dados sem paginação

**Evidência:** `app/avaliacao_api.py:206-249` consulta ativo, comparáveis e liquidez;
  fila usa `LIMIT` controlado apenas pelo cliente (`:253-270`).
**Impacto:** latência e custo crescentes.
**Recomendação:** limites fortes, paginação cursor, métricas SQL e planos `EXPLAIN` em
  homologação.
**Confiança:** alta.

### PERF-003 — MEDIA — Redis declarado sem propósito operacional

**Evidência:** `docker-compose.yml:32-36`, nenhum import de cliente Redis/worker encontrado.
**Impacto:** custo e falsa expectativa de processamento assíncrono; não há retry/backoff.
**Recomendação:** implementar consumidor observável ou retirar até haver caso de uso.
**Confiança:** alta.

### PERF-004 — MEDIA — LSH de imagens é local ao processo

**Evidência:** `app/imagens.py:136-161` usa dicionários em memória.
**Impacto:** perda do índice ao reiniciar e inconsistência entre réplicas.
**Recomendação:** persistir hashes e usar índice/worker compatível com escala.
**Confiança:** alta.

### PERF-005 — MEDIA — backup não é garantia operacional por si só

**Evidência:** `scripts/backup.sh:4-10` grava em caminho relativo e remove bases antigas
  após 14 dias; cron é apenas documentação (`docs/DEPLOY.md:237-248`).
**Impacto:** backup pode ficar no mesmo host, sem retenção externa, sem alerta e sem prova
  de execução.
**Recomendação:** storage externo/imutável, monitoramento, RPO/RTO explícitos e restore
  automatizado em ambiente isolado.
**Confiança:** média; agendamento real não verificável.

### PERF-006 — NÃO VERIFICADO — capacidade real

Não houve carga, `EXPLAIN`, medição de pool, latência de rede, limites de CPU/memória ou
teste de restore. Os números de performance presentes nos comentários do código são
alegações históricas não reproduzidas nesta auditoria.
