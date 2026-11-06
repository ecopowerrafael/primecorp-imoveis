# 01 — Inventário tecnológico

## Estado do checkout

- Branch `main`, commit `dc2c8477189615593a0effb6b74c73b62def18ba`.
- Diretórios: `app`, `db`, `docs`, `migrate`, `nginx`, `node_patch`, `scripts`, `testes`,
  `tests`, `web`.
- Arquivos relevantes: `Dockerfile`, `docker-compose.yml`, `.env.example`, `requirements.txt`.
- Não há `package.json`, `package-lock.json`, `requirements.lock`, `pyproject.toml` ou CI
  visível no checkout atual.

## Stack confirmada por código/configuração

| Camada | Tecnologias |
|---|---|
| API | Python, FastAPI, Pydantic, Uvicorn |
| Dados | PostgreSQL 16, PostGIS 3.4, pg_trgm, citext, pgcrypto |
| Segurança | Argon2id, TOTP/pyotp, cookie de sessão persistido |
| Frontend | HTML/CSS/JS vanilla, motor duplicado em `web/assets/motor.js` |
| Infra | Docker Compose, nginx, Redis declarado |
| Documentos | Markdown; WeasyPrint para apresentação |
| Testes | pytest; Node + jsdom declarado pela documentação |

## Pontos de entrada

- API: `app.main:app` (`app/main.py:39-42`).
- Startup/shutdown: `app/main.py:28-36`.
- Site: `web/index.html`.
- Painel: `web/painel.html`.
- Migração: `migrate/migrar.py:196-227`.
- Usuários: `scripts/cadastrar_equipe.py`, `scripts/criar_usuario.py`.
- Backup/restore: `scripts/backup.sh`, `scripts/restore_test.sh`.
- Testes: `tests/`, `testes/rodar.sh`.

## Dependências e riscos de compatibilidade

As versões estão fixadas em `requirements.txt`, mas sem hash/lock transitive. Não foi
executada auditoria CVE nem instalação, portanto vulnerabilidades de dependências são
**não verificadas**. O frontend de testes usa CommonJS (`require`), enquanto a execução
local foi afetada pelo `package.json` do diretório pai (`type: module`); o repositório não
possui manifesto próprio que estabeleça o contrato correto.

## Legado, duplicação e código sem consumidor confirmado

- `node_patch/CORRECOES_server_mjs.md` é documentação de correções de um portal Node
  anterior, não um serviço executável no checkout.
- `web/assets/motor.js` duplica regras de `app/comparaveis.py` e `app/captacao.py`.
- `Redis` é container declarado, mas não há worker, fila ou importação que o consuma.
- `app/imagens.py` implementa hash/LSH somente em memória; não há rota de upload/indexação.
- `db/v_publicacao_autorizada` existe, mas não há endpoint que a leia.
- Várias tabelas têm schema, mas não possuem handlers de CRUD no backend atual.

## Testes/validações realizadas

- `node --check` passou para os cinco arquivos JS verificados.
- A execução dos testes JS não passou da inicialização: `require` foi rejeitado por
  `type: module` herdado do diretório pai.
- Python/pytest não ficaram disponíveis no ambiente: o launcher `python.exe` falhou ao
  iniciar e `pytest` não foi encontrado.

## Achados

### INV-001 — MEDIA — ausência de lockfiles

**Descrição:** dependências diretas são fixadas, mas transientes não são reproduzíveis.
**Evidência:** `requirements.txt:1-12`; ausência de manifestos/lockfiles na listagem.
**Impacto:** builds podem mudar sem alteração de código e a auditoria de CVE fica incompleta.
**Recomendação:** gerar lock reproducível por ambiente e scanner no CI.
**Confiança:** alta.

### INV-002 — MEDIA — contrato Node ambíguo

**Descrição:** testes CommonJS não têm manifesto no projeto e sofrem configuração do pai.
**Evidência:** `testes/*.test.js` usam `require`; execução observada falhou com ES module.
**Impacto:** o comando documentado não é portátil.
**Recomendação:** adicionar manifesto mínimo de teste com estratégia explícita ou converter
  os testes para ESM, sem depender do diretório pai.
**Confiança:** alta.
