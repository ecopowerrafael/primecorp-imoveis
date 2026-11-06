# 09 — Frontend e UX

## Site

O site tem navegação, combobox de municípios, vitrine e validação básica de formulário.
O campo de consentimento é obrigatório na UI (`web/index.html:392-395`), mas a submissão
não é entregue ao backend (`:758-776`). A consulta de mercado não tem dados configurados
(`web/assets/calibracao.js:30-80`).

## Painel

Há layout responsivo, abas, mensagens e motor local. Porém autenticação, dados de ativos,
conformidade e mandato são fixtures/estado local. A própria página contém textos que dizem
que as chamadas devem ser trocadas por `fetch` quando a API subir (`web/painel.html:1120-1125`).

## Segurança de renderização

Vários resultados entram em `innerHTML` com valores derivados de CSV ou arrays, por exemplo
`web/painel.html:1291-1297`, `:1300-1303` e `:1524-1534`. Hoje esses valores vêm do
input colado localmente, mas se o mesmo padrão receber dados do backend ou CSV malicioso,
há risco de XSS DOM. O CSP da API não protege necessariamente o host estático se servido
separadamente.

## Achados

### UX-001 — CRITICA — interface comunica sucesso sem operação real

**Evidência:** painel troca de tela sem autenticação (`web/painel.html:1075-1088`); site
mostra recibo mesmo sem endpoint (`web/index.html:760-765`).
**Impacto:** decisões e expectativas incorretas.
**Recomendação:** estados explícitos de demonstração/desabilitado ou integração real antes
  de publicar.
**Confiança:** alta.

### UX-002 — ALTA — XSS potencial no motor de apresentação

**Evidência:** interpolação em `innerHTML` de título, portal, cidade e demais campos
  (`web/painel.html:1291-1303`, `:1524-1534`).
**Impacto:** ao conectar fonte externa/CSV real, payload com HTML/JS pode executar no
  contexto do painel.
**Recomendação:** escapar por padrão ou construir nós com `textContent`; sanitização
  allowlist só quando HTML for requisito.
**Confiança:** alta como risco de design; exploração externa depende da integração.

### UX-003 — MEDIA — calibração vazia e dados estáticos

**Evidência:** `calibracao.js:33-38,78-80`; `painel.html:1487-1503`.
**Impacto:** experiência parece operacional, mas não reflete base atual.
**Recomendação:** exibir claramente “sem base”, carregar dados autenticados e mostrar data/
  responsável da calibração.
**Confiança:** alta.

### UX-004 — BAIXA — acessibilidade precisa de validação real

Há atributos ARIA e labels, mas não foi executado leitor de tela, teclado completo,
contraste ou auditoria automatizada. Classificação: **não verificado**.
