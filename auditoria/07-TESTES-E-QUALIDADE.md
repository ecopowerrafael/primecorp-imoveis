# 07 — Testes e qualidade

## Suítes encontradas

- Python: `tests/test_avaliacao.py` e `tests/test_seguranca.py`.
- JavaScript: `testes/motor.test.js`, `site.test.js`, `painel.test.js`.
- Runner: `testes/rodar.sh:7-10`.

Os números publicados divergem entre documentos: README fala em 64/50/27/52; arquitetura
fala em 64/32/27/46; apresentação fala em 193 total. Isso não foi validado por execução.

## Execução desta auditoria

- `node --check` passou para os arquivos JS.
- `node testes/*.test.js` falhou antes dos testes porque `require` não é permitido no modo
  ESM herdado do `package.json` do diretório pai.
- `pytest` não está instalado/disponível e o launcher Python local falhou ao iniciar.
- Não foram instaladas dependências nem modificados testes.

## Cobertura observável

Há boa cobertura de funções puras de avaliação, captação, imagens, segurança e PTAM. Não
há testes de integração FastAPI/PostgreSQL no checkout, apesar de existirem rotas críticas.
Também não há E2E real entre browser/API, autorização por tenant/ownership, concorrência
de PTAM, upload, importação concorrente, idempotência de leads ou restore automatizado.

## Achados

### TEST-001 — CRITICA — caminho crítico não é coberto por teste E2E

**Evidência:** testes JS exercitam HTML/motor local; `web/painel.html` não chama a API
(`:1071-1125`). Não há `TestClient`/fixture PostgreSQL nos testes.
**Impacto:** suíte verde poderia coexistir com login e captação quebrados.
**Recomendação:** testes de contrato e E2E com banco efêmero/homologação: login, MFA,
  `/eu`, lead, avaliação, mandato e 403 cruzado.
**Confiança:** alta.

### TEST-002 — ALTA — runner não é portátil neste checkout

**Evidência:** `testes/rodar.sh:7-10` exige `python3`, jsdom e CommonJS; ambiente observado
  não tem pytest e o contexto Node é ESM.
**Impacto:** documentação promete uma verificação que não reproduz em checkout limpo.
**Recomendação:** manifestos/lockfiles, CI com versões declaradas e comando cross-platform.
**Confiança:** alta.

### TEST-003 — ALTA — invariantes de autorização ausentes

**Evidência:** nenhuma suíte encontrada cobre `app/main.py:266-284` ou o papel faltante
  em `:93-98`.
**Impacto:** regressões de IDOR/RBAC passam despercebidas.
**Recomendação:** matriz parametrizada de roles, ownership, tenant, opt-out e dados nulos.
**Confiança:** alta.

### TEST-004 — MEDIA — concorrência e idempotência não testadas

**Evidência:** não há testes de `count()+1` em `app/avaliacao_api.py:134-140`, duas
  importações simultâneas ou retry de lead/webhook.
**Impacto:** falhas somente em produção sob carga.
**Recomendação:** testes concorrentes com unique constraints e reexecução de requests.
**Confiança:** alta.

### TEST-005 — MEDIA — divergência entre testes e documentação

**Evidência:** contagens diferentes em `README.md:141-146`, `docs/ARQUITETURA.md:182-185`
  e `docs/APRESENTACAO_DESENVOLVEDOR.md:132-135`.
**Impacto:** falsa percepção de cobertura.
**Recomendação:** contar testes automaticamente no CI e remover números manuais.
**Confiança:** alta.
