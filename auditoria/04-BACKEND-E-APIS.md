# 04 — Backend e APIs

## Rotas e controles

As rotas estão catalogadas em `03-FUNCIONALIDADES.md`. A API usa queries parametrizadas,
Pydantic para vários inputs e dependências de sessão/papel. O banco adiciona constraints
e triggers para CRECI e publicidade.

## Pontos positivos confirmados

- Sessões são persistidas por hash e revogáveis (`app/auth.py:235-289`).
- Senhas usam Argon2id via `app/security.py`.
- Importação tem limite de tamanho/linhas e transação única (`app/main.py:199-259`).
- SQL observado usa parâmetros, sem concatenação de input em queries de negócio.
- Erros genéricos não retornam stack trace (`app/main.py:62-65`).

## Achados

### API-001 — ALTA — operação de opt-out sem autorização de escopo

**Evidência:** `app/main.py:266-284` depende somente de `usuario_atual` e atualiza por
`contact_id`; não verifica se o chamador é o titular, proprietário relacionado ou papel
jurídico.
**Cenário:** qualquer sessão válida pode enumerar IDs e marcar contatos de terceiros como
opostos, afetando captação e dados pessoais.
**Impacto:** alteração indevida de dados, DoS de operação comercial e quebra de integridade
da trilha LGPD.
**Correção sugerida:** separar endpoint autenticado de backoffice de endpoint de titular,
exigir prova/fluxo de verificação, papel adequado e ownership/tenant; tornar a operação
idempotente com DSR único.
**Confiança:** alta.

### API-002 — ALTA — criação de mandato sem checagem de papel

**Evidência:** `app/main.py:93-98` usa `Depends(exigir_creci)`, não `exigir_papeis`.
**Impacto:** qualquer conta com CRECI ativo, inclusive papel de leitura ou sem papel
comercial, pode criar mandato.
**Correção:** combinar CRECI, papel `corretor`/`admin` e escopo do ativo/contato.
**Confiança:** alta.

### API-003 — ALTA — ausência de verificação de relação entre ativo, proprietário e usuário

**Evidência:** `app/main.py:107-133` valida o CRECI do responsável e existência/opt-out do
contato, mas não valida que `owner_contact_id` pertence ao `asset_id`, nem que o usuário
tem acesso ao ativo.
**Impacto:** criação de vínculo de negócio incorreto ou entre registros arbitrários.
**Correção:** validar relação em transação e aplicar ownership/tenant.
**Confiança:** alta.

### API-004 — MEDIA — parâmetro `limite` sem validação de faixa

**Evidência:** `app/avaliacao_api.py:253-270` aceita `limite: int = 25` e envia ao SQL.
**Impacto:** valores negativos, muito altos ou zero geram comportamento inesperado e podem
  pressionar banco/CPU.
**Correção:** `Query(ge=1, le=100)` ou limite de serviço centralizado.
**Confiança:** alta.

### API-005 — MEDIA — número de PTAM sujeito a corrida

**Evidência:** `app/avaliacao_api.py:134-140` calcula `count(*) + 1` antes de registrar
  auditoria; não há sequence/unique constraint para número.
**Impacto:** requisições concorrentes podem gerar números duplicados.
**Correção:** sequence PostgreSQL ou tabela de numeração com lock/constraint única e teste
  concorrente.
**Confiança:** alta.

### API-006 — MEDIA — importação lê todo o arquivo em memória e roda no request

**Evidência:** `app/main.py:202-217` faz `read()` e `list(leitor)` até 32 MB/100 mil linhas.
**Impacto:** picos de memória, bloqueio do worker e rollback caro; Redis declarado não ajuda.
**Correção:** streaming/chunks, job assíncrono, status consultável, limites por usuário e
  idempotency key.
**Confiança:** alta.

### API-007 — MEDIA — histórico de preço não identifica o lote importado

**Evidência:** `app/main.py:239-248` seleciona todas as listings com `last_seen_at` na
  janela de um minuto, não apenas fingerprints/IDs desta requisição.
**Impacto:** concorrência entre importações pode registrar preços de outro lote e tornar
  métricas históricas imprecisas.
**Correção:** capturar IDs afetados em tabela temporária/CTE ou usar `import_run_id`.
**Confiança:** alta.
