# Auditoria técnica — resumo executivo

## Escopo e método

Auditoria estática do checkout local em 06/11/2026. Foram lidos código Python, SQL,
HTML/JS, Docker, nginx, scripts, documentação e testes. Não foram instaladas
dependências, iniciados containers, acessados serviços externos ou alterados arquivos
existentes. A única saída criada é esta pasta `auditoria/`.

Estado inicial confirmado: branch `main`, commit `dc2c8477189615593a0effb6b74c73b62def18ba`,
working tree limpa e remoto `origin` apontando para o repositório informado.

## 1. O que o sistema realmente é

É uma base de aplicação para intermediação imobiliária, composta por:

- API FastAPI/Python com autenticação, MFA, RBAC, avaliação por comparáveis, PTAM,
  captação, importação CSV, mandatos, LGPD e consultas operacionais.
- Schema PostgreSQL/PostGIS com regras de CRECI, autorização de publicidade, contatos,
  leads, anúncios, negócios, comissões e auditoria.
- Site público estático com vitrine e consulta preliminar de faixa.
- Painel HTML/JS demonstrativo, com motor de cálculo local e dados de exemplo.

O sistema não é, no estado observado, uma aplicação ponta a ponta pronta para operação.
O backend existe, mas o painel não o consome e o site não persiste leads.

## 2. Tecnologias confirmadas

Python 3.12 declarado; FastAPI 0.115.6; Uvicorn; psycopg 3/pool; PostgreSQL 16 +
PostGIS 3.4, pg_trgm, citext e pgcrypto; Argon2id; TOTP; Pillow; HTML/CSS/JavaScript
sem build; Docker Compose; nginx; Redis declarado; pytest e jsdom declarados na
documentação. Não há `package.json`, lockfile Node ou lockfile Python no repositório.

## 3. Arquitetura funcional atual

```text
navegador -> nginx/TLS/rate limit -> FastAPI -> pool psycopg -> PostgreSQL/PostGIS
     |                 |
     +-> web/index.html (site; formulário sem endpoint)
     +-> web/painel.html (protótipo local; dados estáticos; sem fetch da API)

Redis -> container declarado no compose, sem consumidor/worker identificado
Object storage -> apenas chave prevista em images; integração não implementada
```

## 4. Estado das funcionalidades

| Área | Estado real | Evidência principal |
|---|---|---|
| Login, sessão, Argon2id, TOTP | Backend implementado; painel não integrado | `app/auth.py:166-366`; `web/painel.html:1075-1088` |
| RBAC e CRECI | Parcialmente implementado | `app/auth.py`; triggers em `db/001_schema.sql:336-350` |
| Avaliação/PTAM | Backend implementado; painel calcula localmente | `app/avaliacao_api.py:64-173`; `web/painel.html:1349-1472` |
| Radar/deduplicação | Motor JS local; sem rota backend correspondente | `web/assets/motor.js:327-560` |
| Mandatos | Criação backend; ativação/anexo não expostos por API | `app/main.py:93-138`; painel usa `MANDATOS` fictício |
| Leads públicos | Não persistidos | `web/index.html:758-776`; não existe POST `/api/leads` |
| Importação CSV | Backend implementado, síncrono e transacional | `app/main.py:186-259` |
| LGPD | Modelo e opt-out parcial; governança de acesso insuficiente | `app/main.py:266-289`; `db/001_schema.sql:255-299` |
| Fila/Redis | Não implementada | `docker-compose.yml:32-36`; documentação confirma ausência |
| PDFs | Script de apresentação; PTAM retorna Markdown | `scripts/gerar_apresentacao_pdf.py`; `app/ptam.py:180-254` |

## 5. Backend integrado ao frontend?

Não. O painel não faz chamadas `fetch` para autenticação ou APIs e aceita qualquer
credencial ao clicar em continuar. O site público define `ENDPOINT = ""` e, nesse
caminho, apenas imprime o lead no console. Portanto a aparência de login, radar,
avaliação, oportunidades e ativação não prova operação persistente.

## 6. Persistência e dados reais

O schema prevê PostgreSQL/PostGIS e o código usa essas tabelas. O checkout não contém
dump de banco, arquivos SQLite/JSON de dados, usuários reais ou credenciais. Há dados
fictícios embutidos no painel, inclusive nomes, CRECI de demonstração, imóveis e
credenciais preenchidas (`web/painel.html:381-386`, `:1131-1132`, `:1487-1503`).
`web/assets/calibracao.js:30-80` deixa o mercado vazio de propósito.

## 7. Dez riscos mais importantes

1. **CRITICA R-01 — painel não autentica nem persiste:** login fictício e dados locais.
2. **CRITICA R-02 — leads públicos são descartados:** `ENDPOINT` vazio no site.
3. **ALTA R-03 — opt-out aceita qualquer usuário autenticado para qualquer contato:**
   `app/main.py:266-284`, sem ownership, papel ou verificação do titular.
4. **ALTA R-04 — falta isolamento de tenant:** não há `tenant_id`/organização por linha;
   queries usam apenas IDs globais.
5. **ALTA R-05 — criação de mandato exige CRECI, mas não papel:**
   `app/main.py:93-98` usa apenas `exigir_creci`.
6. **ALTA R-06 — migrações SQL não são versionadas/idempotentes:** scripts de inicialização
   e `004` não têm mecanismo de versão/rollback.
7. **ALTA R-07 — importação aceita arquivo inteiro em memória até 32 MB/100 mil linhas:**
   `app/main.py:199-217`; pode bloquear processo e saturar pool.
8. **MEDIA R-08 — série de preço pode registrar várias linhas no mesmo lote/janela:**
   critério `last_seen_at > now()-1 minute` não identifica apenas as linhas importadas.
9. **MEDIA R-09 — confiança em `X-Forwarded-For` combinada com Uvicorn `--forwarded-allow-ips=*`**:
   superfície de falsificação de IP se a API for exposta fora do proxy.
10. **MEDIA R-10 — painel expõe PII e CRECI fictícios no HTML público do artefato:**
    publicação acidental de `painel.html` contradiz o próprio README.

## 8. Bloqueadores de produção

- Não publicar o painel como ferramenta operacional antes de conectá-lo à API e remover
  credenciais/dados demonstrativos.
- Implementar endpoint público de leads com validação, consentimento, rate limit,
  anti-abuso, persistência e confirmação de recebimento.
- Definir modelo de tenant/organização ou declarar formalmente que o produto é single-
  tenant; hoje o schema não suporta isolamento SaaS.
- Corrigir autorização do opt-out e das operações de intermediação.
- Estabelecer migrations versionadas, backup/restore realmente agendados e homologação.
- Validar dependências e executar testes em ambiente limpo; a suíte não foi executada
  nesta auditoria.

## 9. Dívidas técnicas relevantes

Integração frontend/backend ausente; Redis sem worker; imagens sem armazenamento/index
persistente; APIs sem consumidores; ausência de CRUD para diversas entidades do schema;
documentação descreve mais funcionalidades que as rotas presentes; falta de observabilidade
estruturada; ausência de lockfile e CI visível no checkout; regras de negócio duplicadas
entre Python e JS.

## 10. Matriz P0/P1/P2/P3

| Prioridade | Itens |
|---|---|
| P0 | R-01, R-02, R-03, R-04; impedir operação com aparência de sucesso sem persistência |
| P1 | R-05, R-06, importação/limites, API do painel, autorização por entidade |
| P2 | Worker/Redis, imagens persistentes, observabilidade, contrato de API, UX real |
| P3 | refinamento visual, calibração de mercado, índices avançados e otimizações após medição |

## 11. Roadmap recomendado

1. **Fundação:** escolher single-tenant ou adicionar tenant; definir matriz RBAC/ownership;
   criar migration framework e ambiente de homologação. Aceite: testes negativos de acesso
   cruzado retornam 403/404 e migrations repetidas são seguras.
2. **Integração:** criar contratos OpenAPI para login, `/eu`, leads, ativos, mandatos,
   avaliação e fila; substituir dados hardcoded por `fetch` com sessão expirada tratada.
   Aceite: cada ação visível gera request real, erro é exibido e refresh não perde estado válido.
3. **Captação:** implementar POST público de lead, rate limit, deduplicação/idempotência,
   base legal e auditoria. Aceite: lead aparece no banco uma vez e opt-out bloqueia recontato.
4. **Segurança de domínio:** corrigir autorização por contato/ativo/mandato, validar upload,
   proxy headers e CORS/CSRF. Aceite: matriz de autorização automatizada e logs sem PII desnecessária.
5. **Operação:** worker para tarefas pesadas, limites/paginação, métricas, alertas, backup
   agendado e restore em homologação. Aceite: teste de carga controlado e restore reproduzível.
6. **Go-live:** remover dados demonstrativos, preencher calibração com evidência aprovada,
   revisar textos legais e realizar smoke test de ponta a ponta.

## Confiança e limitações

Alta para achados baseados diretamente em arquivos e linhas. Não verificado: banco em
execução, produção, DNS, certificados, configuração real de proxy, vulnerabilidades de
dependências em runtime e dados históricos fora do checkout.
