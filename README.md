# second-brain-contrib

Caixa de entrada de contribuições pra **vitrine** do template-frontend (golden reference de UI).

**Isto NÃO é uma biblioteca de componentes.** Componente aceito é promovido pro golden pelo mantenedor, propaga pra todo mundo no próximo bundle do share, e a pasta dele é removida daqui. O que mora em `contrib/` é só o que está aguardando review.

## Fluxo

1. Monta o pacote no formato abaixo — na mão ou com a skill `/sb-propor` (já vem no claude-config do share).
2. Abre PR criando `contrib/<slug>/`.
3. Mantenedor revisa → integra no golden → próximo bundle do share entrega pra todo mundo.
4. A pasta sai do repo: PR merged = promovido; PR closed = recusado (motivo fica no PR).

## Formato do pacote

```
contrib/<slug>/
├── <Nome>Page.vue        # tela/componente, escrito como se já morasse na vitrine
├── mocks/<slug>.json     # dado 100% mock
├── i18n/
│   ├── pt-BR.js          # fatia i18n (namespace vitrine.<slug>)
│   ├── en-US.js
│   └── es-ES.js
└── README.md             # handoff pro mantenedor (o que é, deps, rota, como testou)
```

**Um slug só, reusado em tudo**: pasta = nome do mock = namespace i18n = rota sugerida (`/vitrine/<slug>`). Só o `.vue` foge (PascalCase livre, sufixo `Page`). Slug divergente entre as peças = tela quebrada na promoção.

Pacote-exemplo completo: [`contrib/lancamentos/`](contrib/lancamentos/).

## Regras (PR fora disso volta)

- **100% mock** — nenhuma chamada de rede (fetch/axios/api). Mock em JSON local.
- **i18n nos 3 locales** — nada de string hardcoded na tela.
- **Componentes canônicos do template** — usa DataGrid/DataTable/EmptyState/shells existentes; não reinventa.
- **Naming genérico** — sem nome de marca, empresa ou sistema real. Domínio genérico (pedido, produto, lançamento).
- **Imports como se já estivesse na vitrine** — `src/vitrine/mocks/...`, `src/components/shared/...`. Promoção vira cópia de arquivo + rota + merge de i18n.

Detalhe de cada arquivo e checklist: [CONTRIBUTING.md](CONTRIBUTING.md).

## Ver a vitrine antes de submeter

No bundle do share: `templates/sistema/template-frontend` → `npm i` → `quasar dev` → rota `/vitrine`. Confere se o que você quer propor já não existe.
