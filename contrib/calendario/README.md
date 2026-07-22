# calendario — handoff

> Submissão pra vitrine do template-frontend. Slug `calendario` reusado em pasta, mock, namespace i18n e rota (regra do CONTRIBUTING §Slug primeiro).

## 1. O que é

Calendário / agenda de recursos com **três visões sobre a mesma fonte de eventos** — Semana (grid 7 col), Mês (grid 6 semanas) e Timeline (1 dia, uma faixa por recurso com linha do "agora"). Popover de detalhe, filtros rápidos client-side, cor do card por recurso + faixa por status, hachura pra bloqueio/manutenção, responsivo (mobile abre na timeline).

## 2. Arquétipo

Peça de calendário/agenda (arquétipo temporal — não existe equivalente na vitrine hoje). "Recurso" é genérico: aqui salas, mas serve pra pessoas/veículos/equipamentos.

## 3. Deps canônicas usadas

- `src/components/shared/PageHeader.vue`
- `src/components/shared/EmptyState.vue` (timeline sem recursos)
- `src/composables/useNotify.js` (toasts das ações demo)
- Quasar base (`q-card`, `q-btn-toggle`, `q-select`, `q-dialog`, `q-skeleton`, `q-inner-loading`)
- Sem componente de calendário canônico no template → markup próprio (grade/timeline). Badge de estado é inline (o template não tem StatusBadge standalone).

## 4. Rota sugerida

`/vitrine/calendario`. No índice (`IndexPage.vue`), grupo **"Telas (arquétipos)"** — sugestão de card:

```js
{ nome: t('vitrine.index.calendario_nome'), desc: t('vitrine.index.calendario_desc'), icon: 'calendar_month', to: '/vitrine/calendario' },
```

Rótulos sugeridos (adicionar em `vitrine.index` dos 3 locales, se o mantenedor topar o card):
pt-BR `calendario_nome: 'Calendário / Agenda'` · `calendario_desc: 'agenda de recursos — semana, mês e timeline sobre a mesma fonte'`.

## 5. Como testar

1. `CalendarioPage.vue` → `src/vitrine/telas/`
2. `mocks/calendario.json` → `src/vitrine/mocks/`
3. `i18n/{pt-BR,en-US,es-ES}.js` → mesclar namespace `vitrine.calendario` nos 3 arquivos de `src/vitrine/i18n/`
4. Rota em `src/vitrine/routes.js` (children do `VitrineShell`):
   `{ path: 'calendario', component: () => import('src/vitrine/telas/CalendarioPage.vue') }`
5. `quasar dev` → `/vitrine/calendario`. Conferir: as 3 visões, navegação de período, filtros, popover, toggle "Modo edição", light + dark, e o mobile (devtools) caindo na timeline.

**Nota de token**: a tela usa `--state-*` (scheduled/occupied/ended/maintenance/blocked/available + `-soft`/`-text`), que o golden **não** tem. Eles viajam **scoped-local** no `<style>` da tela (light + dark), então funciona drop-in sem tocar o CSS global. Se fizer sentido cross-tela, promover pro `src/css/app.scss` e apagar da tela (decisão do mantenedor).

**Nota de mock**: `calendario.json` traz eventos por **offset** (`diaOffset` + `inicio`/`fim` "HH:MM", ou `ancora: "agora"`), montados sobre o relógio no load — assim a demo fica sempre populada e sempre tem um evento em andamento. Continua 100% mock, sem rede.

## 6. Como foi testado

Colado na vitrine local (golden `template-frontend`, `npm i`) e **compilado por production build verde** (`quasar build` — Quasar 2.19.3 / @quasar/app-vite 2.6 / Vite 8). A rota `/vitrine/calendario` emitiu seus próprios chunks (`CalendarioPage-*.js` 24.9 KB, `CalendarioPage-*.css` 12.5 KB) com **todos os imports resolvidos** (PageHeader, EmptyState, useNotify, `calendario.json`) e i18n mesclado nos 3 locales; o card no índice renderiza (verificado na árvore de acessibilidade do dev server).

O build pegou 2 bugs reais, **já corrigidos**: doc-comment do `<script>` fechado com `-->` em vez de `*/`, e um `*/` acidental dentro de um comentário CSS (`--text-*/…`).

**Não exercitado interativamente no browser** (clicar as 3 visões, dark/mobile): o browser interno não cooperou com o dev server (screenshot travando por HMR, navegação client-side não disparando). Recomendo ao revisor rodar `quasar dev` e conferir visualmente as 3 visões + light/dark + mobile antes do merge.
