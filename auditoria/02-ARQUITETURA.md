# 02 — Arquitetura real

## Fluxo

```text
Site público (HTML/JS)
  ├─ consulta de faixa -> tabela local CALIBRACAO.MERCADO (vazia)
  └─ formulário -> ENDPOINT vazio -> console.info

Painel (HTML/JS)
  ├─ login fictício -> apenas troca de telas
  ├─ radar/anúncios/avaliação/oportunidades -> Motor.js + arrays locais
  └─ mandato -> objetos locais MANDATOS

API FastAPI
  ├─ auth -> sessions/users/roles/login_attempts
  ├─ avaliação/PTAM/captação -> listings/assets/audit_log
  ├─ mandatos/pipeline/LGPD/CRECI -> views e tabelas PostgreSQL
  └─ importação CSV -> listings/import_runs/price_history

PostgreSQL/PostGIS
  └─ schema inicial 001..004; sem tenant_id
```

## Responsabilidades

`app/main.py` concentra ciclo de vida, headers, saúde, mandatos, importação e LGPD.
`app/auth.py` concentra autenticação, sessão e papéis. `app/avaliacao_api.py` expõe
avaliação, PTAM e fila. `app/comparaveis.py`, `captacao.py`, `ptam.py` e `imagens.py`
contêm domínio. O banco contém parte substancial das regras regulatórias.

## Acoplamento e duplicação

Há uma decisão válida de manter frontend sem build, mas o domínio foi portado para JS
sem contrato automatizado contra a API. A mesma regra de avaliação existe no backend e
no `Motor.js`; os testes pretendem comparar casos, mas a integração real não existe.

## Pontos únicos de falha

- PostgreSQL concentra sessão, negócio, LGPD e auditoria.
- API é síncrona; tarefas pesadas não têm worker.
- nginx e certificados são obrigatórios para segurança declarada, mas não há validação
  de configuração no repositório.
- O site público não possui caminho funcional de persistência de lead.

## Achados

### ARC-001 — CRITICA — duas aplicações divergentes sob a mesma UI

**Evidência:** painel local em `web/painel.html:1071-1125`; endpoints backend em
`app/avaliacao_api.py:64-271`. Não há `fetch` no painel para esses endpoints.
**Impacto:** usuário pode tomar decisão com dados fictícios e acreditar que a operação foi
gravada. É falha de integridade operacional, não apenas de UX.
**Recomendação:** contrato de API e integração progressiva por módulo; bloquear rótulo
operacional até cada ação ter resposta persistida.
**Confiança:** alta.

### ARC-002 — ALTA — sem fronteira de tenant

**Evidência:** schema de negócio não possui `tenant_id` (`db/001_schema.sql:22-432`);
queries operacionais filtram por `id`/atributos globais, por exemplo `app/main.py:115` e
`app/avaliacao_api.py:206`.
**Impacto:** não há isolamento técnico para transformar o produto em SaaS multi-tenant;
um usuário autorizado pode consultar registros fora de seu escopo se novos tenants forem
adicionados por configuração externa.
**Recomendação:** decidir single-tenant explicitamente ou introduzir tenant, FKs, escopo
de sessão, índices e testes de acesso cruzado antes de produção multiempresa.
**Confiança:** alta.

### ARC-003 — ALTA — domínio de leads não fecha o fluxo

**Evidência:** tabela `leads` existe em `db/001_schema.sql:398-416`, mas não há rota de
criação; site usa `ENDPOINT = ""` em `web/index.html:758-765`.
**Impacto:** principal conversão do site não chega ao banco.
**Recomendação:** implementar endpoint e ligar o site com contrato e idempotência.
**Confiança:** alta.
