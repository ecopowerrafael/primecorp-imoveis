# 03 — Inventário funcional

## Site público

| Funcionalidade | Arquivos | Estado | Evidência/riscos |
|---|---|---|---|
| Vitrine de imóveis | `web/index.html:625-631` | Simulada | array `IMOVEIS` local; não consulta `v_publicacao_autorizada` |
| Consulta preliminar de faixa | `web/index.html:679-727`, `calibracao.js:30-80` | Parcial | tabela vazia; modo API previsto, sem endpoint implementado |
| Formulário de contato | `web/index.html:731-776` | Quebrada em produção | endpoint vazio; só `console.info` |
| Consentimento visual | `web/index.html:392-395` | Parcial | checkbox sem persistência, versão de aviso ou IP/artefato |

## Painel

| Módulo | Estado | Evidência |
|---|---|---|
| Login/MFA | Simulado | `web/painel.html:1075-1088` não chama API |
| Radar | Implementado somente no motor local | `web/painel.html:1137-1237`, `motor.js:327-552` |
| Anúncios/deduplicação | Motor local | `web/painel.html:1261-1328` |
| Avaliação | Motor local; backend separado | `web/painel.html:1349-1401`; API `app/avaliacao_api.py:64-89` |
| PTAM | Texto local com número fixo `0001` | `web/painel.html:1403-1472` |
| Oportunidades | Array local `CANDIDATOS` | `web/painel.html:1487-1575` |
| Mandato | Estado local; sem upload/API | `web/painel.html:1577-1619` |
| Conformidade | Conteúdo estático | `web/painel.html:988-1065` |

## Backend/API

| Método/rota | Auth | Dados | Estado |
|---|---|---|---|
| POST `/api/auth/login` | público + senha/MFA | users/sessions | implementado |
| POST `/api/auth/logout`, `/logout-todas` | sessão | sessions | implementado |
| GET `/api/auth/eu` | sessão | users/roles | implementado |
| POST `/api/auth/mfa/setup`, `/confirmar`, `/senha` | sessão | users | implementado, requer revisão operacional |
| GET `/api/health` | público | PostGIS | implementado; divulga versão/horário |
| POST `/api/mandatos` | CRECI | mandates/contacts | implementado, autorização de papel insuficiente |
| GET `/api/mandatos/vencendo` | papel | view | implementado |
| GET `/api/pipeline/comissao` | papel | view | implementado |
| POST `/api/import/csv` | originacao/admin | listings/history | implementado, síncrono |
| POST `/api/avaliacao` | papel | listings/assets | implementado |
| POST `/api/ptam` | CRECI | assets/audit | implementado; sequência concorrente frágil |
| POST `/api/captacao/avaliar` | papel | assets/listings | implementado |
| GET `/api/captacao/fila` | papel | assets/mandates | implementado; `limite` sem validação |
| POST `/api/lgpd/opt-out/{id}` | qualquer sessão | contacts/DSR | autorização insuficiente |
| GET expurgo/CRECI | jurídico/admin | views | implementado |

## Funcionalidades não encontradas

Não foram localizados endpoints/handlers funcionais para CRUD de usuários/roles, leads
públicos, ativos, contatos, imóveis públicos, upload de autorização, ativação de mandato,
negócios, comissões, documentos, imagens, geocodificação, exportações, auditoria de
consulta, anonimização/expurgo efetivo, webhooks ou worker Redis.

## Achados

### FUN-001 — CRITICA — sucesso falso no login do painel

**Evidência:** campos preenchidos com e-mail/senha em `web/painel.html:381-386` e função
`entrar()` apenas troca `hidden` em `:1084-1088`.
**Impacto:** controle de acesso visualmente enganoso; qualquer pessoa com o HTML acessa
dados demonstrativos e o usuário não sabe que não houve autenticação.
**Recomendação:** integrar `/api/auth/login`, remover valores default e bloquear módulos
até `/api/auth/eu` confirmar sessão.
**Confiança:** alta.

### FUN-002 — CRITICA — formulário público não entrega lead

**Evidência:** `web/index.html:758-765`.
**Impacto:** perda silenciosa de oportunidades e possível descumprimento da promessa
"Recebido".
**Recomendação:** endpoint real, status de entrega distinto de sucesso visual e teste E2E.
**Confiança:** alta.

### FUN-003 — MEDIA — PTAM local não é documento oficial do backend

**Evidência:** número fixo no painel `web/painel.html:1411`; backend gera sequência em
`app/avaliacao_api.py:134-173`.
**Impacto:** duplicidade de numeração e emissão sem trilha persistida.
**Recomendação:** emitir exclusivamente pela API e retornar documento/ID persistido.
**Confiança:** alta.
