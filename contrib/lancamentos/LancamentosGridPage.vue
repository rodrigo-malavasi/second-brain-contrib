<template>
  <q-page padding class="column q-gutter-md">
    <div>
      <div class="text-h6">{{ t('vitrine.lancamentos.titulo') }}</div>
      <div class="text-caption text-grey-7">{{ t('vitrine.lancamentos.subtitulo') }}</div>
    </div>

    <DataGrid
      :rows="lancamentos"
      :columns="colunas"
      row-key="id"
      @add-row="adicionarLinha"
    >
      <template #no-data>
        <EmptyState
          variant="empty"
          icon="receipt_long"
          :title="t('vitrine.lancamentos.vazioTitulo')"
          :description="t('vitrine.lancamentos.vazioDesc')"
        />
      </template>

      <!-- coluna calculada (read-only) -->
      <template #body-cell-subtotal="props">
        <q-td :props="props" class="text-right text-weight-medium">
          {{ moeda(subtotalLinha(props.row)) }}
        </q-td>
      </template>

      <!-- ações por linha -->
      <template #header-cell-acoes="props">
        <q-th :props="props"><span class="sr-only">{{ t('vitrine.lancamentos.remover') }}</span></q-th>
      </template>
      <template #body-cell-acoes="props">
        <q-td :props="props" class="text-center">
          <q-btn
            flat dense round size="sm" icon="delete"
            :aria-label="t('vitrine.lancamentos.remover')"
            @click="lancamentos.splice(props.rowIndex, 1)"
          />
        </q-td>
      </template>

      <!-- rodapé com total -->
      <template #bottom-row>
        <q-tr>
          <q-td colspan="4" class="text-right text-weight-bold">{{ t('vitrine.lancamentos.total') }}</q-td>
          <q-td class="text-right text-weight-bold">{{ moeda(total) }}</q-td>
          <q-td />
        </q-tr>
      </template>
    </DataGrid>
  </q-page>
</template>

<script setup>
// LANÇAMENTOS = demo do DataGrid com os 4 editores de célula (select, number, currency,
// percent) + coluna calculada em slot + ações por linha + total no bottom-row + add-row
// na borda (Enter na última célula cria linha nova; Tab é circular, ↓ para).
import { ref, computed } from 'vue';
import { useI18n } from 'vue-i18n';
import DataGrid from 'src/components/shared/grid/DataGrid.vue';
import EmptyState from 'src/components/shared/EmptyState.vue';
import dados from 'src/vitrine/mocks/lancamentos.json';

const { t, locale } = useI18n();

const lancamentos = ref(structuredClone(dados.lancamentos));
let seq = 0;

const colunas = computed(() => [
  {
    name: 'categoria', label: t('vitrine.lancamentos.colCategoria'), field: 'categoria', align: 'left',
    editor: { type: 'select', options: dados.categorias },
  },
  {
    name: 'qtd', label: t('vitrine.lancamentos.colQtd'), field: 'qtd', align: 'right',
    editor: { type: 'number', min: 0 },
  },
  {
    name: 'valorUnit', label: t('vitrine.lancamentos.colValorUnit'), field: 'valorUnit', align: 'right',
    editor: { type: 'currency' },
  },
  {
    name: 'descontoPct', label: t('vitrine.lancamentos.colDesconto'), field: 'descontoPct', align: 'right',
    editor: { type: 'percent' },
  },
  { name: 'subtotal', label: t('vitrine.lancamentos.colSubtotal'), field: 'subtotal', align: 'right' },
  { name: 'acoes', label: '', field: 'acoes', align: 'center' },
]);

const subtotalLinha = (r) => (r.qtd || 0) * (r.valorUnit || 0) * (1 - (r.descontoPct || 0) / 100);
const total = computed(() => lancamentos.value.reduce((s, r) => s + subtotalLinha(r), 0));

const moeda = (v) =>
  new Intl.NumberFormat(locale.value, { style: 'currency', currency: 'BRL' }).format(v);

function adicionarLinha() {
  lancamentos.value.push({ id: `novo-${++seq}`, categoria: null, qtd: 1, valorUnit: 0, descontoPct: 0 });
}
</script>
