# Como contribuir

## Antes de começar

1. Roda a vitrine local (share → `templates/sistema/template-frontend` → `npm i` → `quasar dev` → `/vitrine`) e confere se o componente/arquétipo já não existe.
2. Se existe parecido e você quer **mudar** o existente: abre issue discutindo, não PR com cópia alterada.

## O pacote, arquivo por arquivo

### `<Nome>Page.vue`

- Escrito no idioma da vitrine: `<script setup>`, Composition API, `useI18n`, componentes canônicos do template.
- Imports com path FINAL (como se o arquivo já estivesse em `src/vitrine/telas/`): mock em `src/vitrine/mocks/<slug>.json`, shared em `src/components/shared/...`.
- Doc-comment no topo do script dizendo o que a tela demonstra.
- Zero chamada de rede. Zero import de store/api real.

### `mocks/<slug>.json`

- Dado plausível de domínio genérico, meia dúzia de registros basta.
- Nomes/valores inventados — nada extraído de sistema real.

### `i18n/{pt-BR,en-US,es-ES}.js`

- Cada arquivo exporta a fatia do namespace: `export default { vitrine: { <slug>: { ... } } }`.
- As 3 traduções completas — chave faltando num locale = PR volta.

### `README.md` (handoff)

Responde, nessa ordem:

1. **O que é** — 1-2 linhas.
2. **Arquétipo** — grid editável / consulta / form / dashboard / peça avulsa.
3. **Deps canônicas usadas** — quais componentes do template importa.
4. **Rota sugerida** — path em `/vitrine/...` + entrada no índice.
5. **Como testar** — onde colar cada arquivo (tela → `src/vitrine/telas/`, mock → `src/vitrine/mocks/`, i18n → merge nos 3 arquivos de `src/vitrine/i18n/`) + rota em `src/vitrine/routes.js`.
6. **Como foi testado** — o que você rodou/viu antes de submeter.

## Checklist do PR

- [ ] Pasta única `contrib/<slug>/` — nada fora dela
- [ ] 100% mock, sem chamada de rede
- [ ] i18n completo nos 3 locales
- [ ] Componentes canônicos (não reinventou tabela/grid/empty-state)
- [ ] Naming genérico, sem marca/sistema/empresa real
- [ ] Imports com path final da vitrine
- [ ] README de handoff completo (6 seções)
- [ ] Testado colando na vitrine local

## Depois do merge

O mantenedor integra no golden, o componente sai no próximo bundle do share, e a pasta é removida daqui. Crédito fica no histórico do PR.
