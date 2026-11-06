# 05 — Banco e persistência

## Modelo

O schema cria extensões PostGIS/pg_trgm/citext/pgcrypto (`db/001_schema.sql:5-10`) e
entidades para identidade, sessões, ativos, listings, histórico de preço, evidências,
imagens, contatos/LGPD, mandatos, negócios, comissões, leads e auditoria.

Há FKs e índices relevantes, incluindo cidade/status, tipo/área, listings ativos,
histórico e auditoria. Triggers atualizam `updated_at` e travam CRECI/publicidade.

## Dados reais

Não há banco, dump ou seed de negócio real no checkout. `db/002_seed_and_views.sql`
semeia apenas papéis (`roles`). Os dados do painel são fictícios e não são persistência.

## Migrations/deploy

Os quatro SQLs são montados como scripts de inicialização Docker (`docker-compose.yml:9-10`).
Não há tabela de versão, ferramenta de migration, rollback ou compatibilidade expand/contract.
`db/003...` usa `CREATE TYPE` sem guarda (`:20`) e `db/004...` usa `ALTER TABLE` direto.

## Achados

### DB-001 — ALTA — inicialização SQL não é migration de produção

**Evidência:** Compose monta `./db` em `/docker-entrypoint-initdb.d` (`docker-compose.yml:9-10`);
DDL em `db/001-004` não tem controle de versão operacional.
**Impacto:** scripts só rodam em volume novo; mudanças posteriores não são aplicadas de
forma controlada. Recriar volume é destrutivo.
**Recomendação:** adotar ferramenta de migrations, tabela de versão, scripts reversíveis
quando possível e política expand/contract.
**Confiança:** alta.

### DB-002 — ALTA — inexistência de tenant no modelo

**Evidência:** todas as entidades principais usam apenas PK global; `organizacao` força
uma única linha (`db/003_organizacao_e_publicidade.sql:22-40`).
**Impacto:** impossibilidade de isolamento multiempresa e risco de vazamento se a aplicação
for reutilizada para mais de uma operação.
**Recomendação:** decisão arquitetural explícita; se SaaS, tenant em usuários e entidades,
FKs, índices compostos, RLS opcional e testes cruzados.
**Confiança:** alta.

### DB-003 — MEDIA — inexistência de constraints para invariantes financeiras/deal

**Evidência:** `deals` permite `closed_price`/`closed_at` sem regra de coocorrência e
`commissions` aceita múltiplas linhas por `deal_id` (`db/001_schema.sql:356-390`).
**Impacto:** pipeline/comissão pode divergir, dificultando conciliação.
**Recomendação:** definir invariantes de estágio, preço/data, unicidade ou versionamento de
comissão e transições auditadas.
**Confiança:** média; depende da regra de negócio pretendida.

### DB-004 — MEDIA — expurgo LGPD é somente consulta

**Evidência:** existe view de pendências em `db/002_seed_and_views.sql:51-57` e endpoint
GET em `app/main.py:287-289`, mas não há job/DELETE/anonymization.
**Impacto:** retenção vencida pode permanecer indefinidamente.
**Recomendação:** workflow jurídico com anonimização/eliminação segura, retenção de auditoria
separada, execução idempotente e teste de restauração/retensão.
**Confiança:** alta.

### DB-005 — MEDIA — imagens sem persistência de conteúdo/indexação

**Evidência:** `images.object_key` prevê storage (`db/001_schema.sql:233-244`), enquanto
`app/imagens.py:136-161` mantém índice em memória e não há endpoint de upload.
**Impacto:** recurso não fecha ciclo e não escala entre processos.
**Recomendação:** storage privado, validação de MIME/tamanho, antivírus quando aplicável,
hash persistido e índice/worker adequado.
**Confiança:** alta.
