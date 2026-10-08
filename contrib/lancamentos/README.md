# lancamentos — handoff

> Pacote-exemplo do formato de submissão. Serve de modelo: copie a estrutura, troque o conteúdo. Slug `lancamentos` reusado em pasta, mock, namespace i18n e rota — essa consistência é regra (CONTRIBUTING §Slug primeiro).

## 1. O que é

Tela de lançamentos com grid editável célula a célula — demonstra os 4 editores do `DataGrid` (select, number, currency, percent), coluna calculada, remoção por linha, total no rodapé e criação de linha pela borda (Enter na última célula; Tab é circular na linha e ↓ para na borda).

## 2. Arquétipo

Grid editável (peça, não CRUD completo).

## 3. Deps canônicas usadas

- `src/components/shared/grid/DataGrid.vue` (+ editores Grid* que ele resolve sozinho)
- `src/components/shared/EmptyState.vue`

## 4. Rota sugerida

`/vitrine/lancamentos`, grupo "Peças (catálogo)" no índice da vitrine.

## 5. Como testar

1. `LancamentosGridPage.vue` → `src/vitrine/telas/`
2. `mocks/lancamentos.json` → `src/vitrine/mocks/`
3. `i18n/*.js` → mesclar namespace `vitrine.lancamentos` nos 3 arquivos de `src/vitrine/i18n/`
4. Rota em `src/vitrine/routes.js` (children do `VitrineShell`):
   `{ path: 'lancamentos', component: () => import('src/vitrine/telas/LancamentosGridPage.vue') }`

## 6. Como foi testado

Compilado pelo Vite da vitrine local (tela + mock colados no template, módulo servido sem erro — sintaxe e imports validados). Não exercitado manualmente no browser.

> Ao usar este pacote como base: rode a tela na vitrine e descreva AQUI o que você viu (edição, teclado, locales). Declarar teste que não rodou = PR volta.
