<template>
  <q-page padding class="cal">
    <PageHeader :title="t('vitrine.calendario.titulo')" :subtitle="t('vitrine.calendario.subtitulo')">
      <template #actions>
        <!-- Toggle demo: simula permissão de escrita (canWrite) pra ver os dois estados da tela -->
        <q-toggle v-model="canWrite" color="primary" :label="t('vitrine.calendario.modoEdicao')" class="q-mr-sm" />
        <q-btn
          flat round dense color="primary" icon="refresh"
          :loading="loading" :aria-label="t('vitrine.calendario.atualizar')"
          @click="fetchFeed(true)"
        >
          <q-tooltip>{{ atualizadoLabel || t('vitrine.calendario.atualizar') }}</q-tooltip>
        </q-btn>
        <q-btn
          v-if="canWrite"
          color="primary" icon="add" unelevated no-caps
          :label="t('vitrine.calendario.novo')" class="q-ml-sm"
          @click="demo('novo')"
        />
      </template>
    </PageHeader>

    <div class="cal__grid">
      <div class="cal__main">
        <!-- Filtros rápidos (client-side sobre o feed já carregado) -->
        <q-card flat bordered class="cal__card">
          <div v-if="isMobile" class="cal__filtros-bar">
            <q-btn flat no-caps color="primary" icon="tune" :label="t('vitrine.calendario.filtros')" @click="filtrosAbertos = !filtrosAbertos">
              <q-badge v-if="filtrosAtivos" color="primary" :label="filtrosAtivos" floating />
            </q-btn>
            <q-btn v-if="filtrosAtivos" flat dense no-caps color="primary" :label="t('vitrine.calendario.limpar')" @click="limparFiltros" />
          </div>
          <q-slide-transition>
            <div v-show="!isMobile || filtrosAbertos" class="cal__filters">
              <q-select
                v-model="recursoIdFilter" dense outlined emit-value map-options
                :label="t('vitrine.calendario.recurso')" :options="recursoFilterOptions" class="cal__filter"
              />
              <q-select
                v-model="capacidadeFilter" dense outlined emit-value map-options
                :label="t('vitrine.calendario.capacidade')" :options="capacidadeOptions" class="cal__filter"
              />
              <q-select
                v-model="statusFilter" dense outlined emit-value map-options
                :label="t('vitrine.calendario.status')" :options="statusFilterOptions" class="cal__filter"
              />
              <q-toggle v-model="minhas" color="primary" :label="t('vitrine.calendario.minhas')" class="cal__ftoggle" />
              <q-btn flat no-caps color="primary" :label="t('vitrine.calendario.limpar')" :disable="!filtrosAtivos" @click="limparFiltros" />
            </div>
          </q-slide-transition>
        </q-card>

        <!-- Calendário -->
        <q-card flat bordered class="cal__card cal__calcard">
          <div class="cal__calhead">
            <q-btn-toggle v-model="view" :options="viewOpts" no-caps unelevated toggle-color="primary"
              color="white" text-color="primary" dense class="cal__seg" />
          </div>

          <div class="cal__navrow">
            <q-btn outline dense no-caps color="primary" :label="t('vitrine.calendario.hoje')" @click="irHoje" />
            <q-btn outline dense color="primary" icon="chevron_left" :aria-label="t('vitrine.calendario.anterior')" @click="navegar(-1)" />
            <q-btn outline dense color="primary" icon="chevron_right" :aria-label="t('vitrine.calendario.proxima')" @click="navegar(1)" />
            <span class="cal__datelabel">{{ dateLabel }}</span>
            <div class="cal__legend">
              <span v-for="l in legend" :key="l.state" class="cal__legdot">
                <i :style="{ background: `var(--state-${l.state})` }" />{{ l.label }}
              </span>
            </div>
          </div>

          <div v-if="loading && !carregouUmaVez" class="cal__skel">
            <q-skeleton type="rect" height="420px" />
          </div>

          <template v-else>
            <!-- Visão Semana -->
            <div v-if="view === 'semana'" class="cal__week">
              <div v-for="(d, i) in week" :key="i" class="cal__daycol">
                <div class="cal__dayhead" :class="{ 'is-weekend': i === 0 || i === 6 }">
                  <span class="cal__wd" :class="{ 'is-today': d.today }">{{ d.wd }}</span>
                  <span class="cal__daynum" :class="{ 'is-today': d.today }">{{ d.day }}</span>
                </div>
                <div class="cal__daybody" :class="{ 'is-clickable': canWrite }" @click.self="novaNoDia(d.date)">
                  <button v-for="(x, j) in d.events" :key="`${x.ev.tipo}-${x.ev.id}-${j}`" class="cal__ev"
                    :class="[cardStatusClass(x.ev), { 'is-hatch': x.ev.tipo === 'bloqueio' }]" @click="openEvent(x.ev)"
                    :style="cardStyle(x.ev)">
                    <span class="cal__ev-time" :style="{ color: cardTimeColor(x.ev) }">{{ x.hInicio }}–{{ x.hFim }}</span>
                    <span class="cal__ev-title">{{ x.ev.titulo }}</span>
                    <span class="cal__ev-room">{{ recursoNome(x.ev.recursoId) }}</span>
                  </button>
                </div>
              </div>
            </div>

            <!-- Visão Mês -->
            <div v-else-if="view === 'mes'" class="cal__month">
              <div class="cal__month-dows">
                <span v-for="w in monthDows" :key="w">{{ w }}</span>
              </div>
              <div v-for="(row, wi) in monthWeeks" :key="wi" class="cal__month-week">
                <div v-for="(c, di) in row" :key="di" class="cal__month-cell" :class="{ 'is-out': !c.inMonth, 'is-weekend': c.weekend }">
                  <div class="cal__month-daynum" :class="{ 'is-today': c.today }">{{ c.day }}</div>
                  <button v-for="(x, k) in c.events.slice(0, 3)" :key="`${x.ev.tipo}-${x.ev.id}-${k}`" class="cal__month-ev"
                    :class="[cardStatusClass(x.ev), { 'is-hatch': x.ev.tipo === 'bloqueio' }]" @click="openEvent(x.ev)"
                    :style="cardStyle(x.ev)">
                    <span class="cal__month-ev-t" :style="{ color: cardTimeColor(x.ev) }">{{ x.hInicio }}</span> {{ x.ev.titulo }}
                  </button>
                  <button v-if="c.events.length > 3" class="cal__month-more" @click="verDia(c.date)">
                    +{{ c.events.length - 3 }} {{ t('vitrine.calendario.mais') }}
                  </button>
                </div>
              </div>
            </div>

            <!-- Visão Timeline (1 dia · faixa por recurso) -->
            <div v-else class="cal__timeline">
              <div class="cal__tl-scroll">
                <div class="cal__tl-inner">
                  <div class="cal__tl-header">
                    <div class="cal__tl-hlabel">{{ t('vitrine.calendario.recursos') }}</div>
                    <div class="cal__tl-hhours">
                      <span v-for="h in hoursTl" :key="h" :style="{ left: pct(h) }">{{ String(h).padStart(2, '0') }}h</span>
                    </div>
                  </div>
                  <div v-for="r in tlRooms" :key="r.id" class="cal__tl-row2">
                    <div class="cal__tl-room2">
                      <span class="cal__tl-dot" :style="{ background: `var(--state-${r.dayState})` }" />
                      <div>
                        <div class="cal__tl-rname">{{ r.nome }}</div>
                        <div class="cal__tl-rsub">{{ r.capacidade }} {{ t('vitrine.calendario.lug') }} · {{ r.unidade?.nome }}</div>
                      </div>
                    </div>
                    <div class="cal__tl-track2" :class="{ 'is-clickable': canWrite }" @click="onTrackClick(r, $event)">
                      <div v-for="h in hoursTl" :key="h" class="cal__tl-gridline" :style="{ left: pct(h) }" />
                      <div v-if="nowVisible" class="cal__tl-now" :style="{ left: nowLeft }" />
                      <button v-for="(b, bi) in r.blocks" :key="`${b.ev.tipo}-${b.ev.id}-${bi}`" class="cal__tl-block2"
                        :class="[cardStatusClass(b.ev), { 'is-hatch': b.ev.tipo === 'bloqueio' }]" :style="tlBlockStyle(b)"
                        @click="openEvent(b.ev)">
                        <span class="cal__tl-btime" :style="{ color: cardTimeColor(b.ev) }">{{ b.hInicio }}–{{ b.hFim }}</span>
                        <span class="cal__tl-btitle">{{ b.ev.titulo }}</span>
                      </button>
                    </div>
                  </div>
                  <EmptyState v-if="!tlRooms.length" icon="event_busy" :title="t('vitrine.calendario.semRecursos')" />
                </div>
              </div>
            </div>
          </template>

          <div class="cal__calfoot">
            <span>{{ totalLabel }}</span>
            <span class="cal__updated">{{ atualizadoLabel }}</span>
          </div>

          <q-inner-loading :showing="loading && carregouUmaVez">
            <q-spinner size="md" color="primary" />
          </q-inner-loading>
        </q-card>
      </div>

      <!-- Painel lateral: agenda de hoje + recursos em destaque (derivados do feed carregado) -->
      <div class="cal__side">
        <q-card flat bordered class="cal__card cal__pad">
          <div class="cal__card-title">{{ t('vitrine.calendario.agendaDoDia') }}</div>
          <div class="cal__card-sub">{{ agendaDoDiaLabel }}</div>
          <div v-if="agendaDoDia.length" class="cal__dayagenda">
            <button v-for="(x, i) in agendaDoDia" :key="`${x.ev.tipo}-${x.ev.id}-${i}`" class="cal__da-row" @click="openEvent(x.ev)">
              <span class="cal__da-dot" :style="{ background: x.ev.tipo === 'bloqueio' ? `var(--state-${stateOf(x.ev)})` : corRecurso(recursosById[x.ev.recursoId]) }" />
              <div class="cal__da-body">
                <div class="cal__da-time">{{ x.hInicio }}–{{ x.hFim }}</div>
                <div class="cal__da-title">{{ x.ev.titulo }}</div>
                <div class="cal__da-sub">{{ recursoNome(x.ev.recursoId) }}</div>
              </div>
              <span class="cal__badge" :style="badgeStyle(stateOf(x.ev))">
                <i :style="{ background: `var(--state-${stateOf(x.ev)})` }" />{{ stateLabelOf(x.ev) }}
              </span>
            </button>
          </div>
          <div v-else class="cal__side-empty">
            {{ hojeNoRange ? t('vitrine.calendario.diaSemEventos') : t('vitrine.calendario.diaForaPeriodo') }}
          </div>
        </q-card>

        <q-card flat bordered class="cal__card cal__pad">
          <div class="cal__card-title">{{ t('vitrine.calendario.recursosDestaque') }}</div>
          <div class="cal__featured">
            <div v-for="r in recursosDestaque" :key="r.id" class="cal__feat-row">
              <span class="cal__feat-swatch" :style="{ background: r.cor }" />
              <div class="cal__feat-body">
                <div class="cal__feat-name">{{ r.nome }}</div>
                <div class="cal__feat-sub">
                  {{ r.capacidade }} {{ t('vitrine.calendario.lug') }} ·
                  {{ r.count }} {{ r.count === 1 ? t('vitrine.calendario.eventoHoje') : t('vitrine.calendario.eventosHoje') }}
                </div>
              </div>
              <span class="cal__badge" :style="badgeStyle(r.dayState)">
                <i :style="{ background: `var(--state-${r.dayState})` }" />{{ dayStateLabel(r.dayState) }}
              </span>
            </div>
            <div v-if="!recursosDestaque.length" class="cal__side-empty">{{ t('vitrine.calendario.semRecursos') }}</div>
          </div>
        </q-card>
      </div>
    </div>

    <!-- Popover de detalhe do evento -->
    <q-dialog v-model="detailOpen" position="right">
      <q-card v-if="selected" class="cal__detail">
        <div class="cal__detail-strip" :style="{ background: `var(--state-${selected.state})` }" />
        <div class="cal__detail-body">
          <div class="cal__detail-head">
            <h3 class="cal__detail-title">{{ selected.titulo }}</h3>
            <q-btn flat round dense size="sm" icon="close" v-close-popup />
          </div>
          <div class="cal__detail-badge">
            <span class="cal__badge" :style="badgeStyle(selected.state)">
              <i :style="{ background: `var(--state-${selected.state})` }" />{{ selected.stateLabel }}
            </span>
          </div>
          <div class="cal__detail-row"><q-icon name="meeting_room" size="16px" /><div><div class="cal__dk">{{ t('vitrine.calendario.recurso') }}</div><div class="cal__dv">{{ selected.recurso }}</div></div></div>
          <div class="cal__detail-row"><q-icon name="event" size="16px" /><div><div class="cal__dk">{{ t('vitrine.calendario.data') }}</div><div class="cal__dv">{{ selected.dataLabel }}</div></div></div>
          <div class="cal__detail-row"><q-icon name="schedule" size="16px" /><div><div class="cal__dk">{{ t('vitrine.calendario.horario') }}</div><div class="cal__dv cal__mono">{{ selected.horario }}</div></div></div>
          <div v-if="selected.responsavel" class="cal__detail-row"><q-icon name="person" size="16px" /><div><div class="cal__dk">{{ t('vitrine.calendario.responsavel') }}</div><div class="cal__dv">{{ selected.responsavel }}</div></div></div>
          <div v-if="selected.tipo === 'evento' && canWrite" class="cal__detail-actions">
            <q-btn outline no-caps color="primary" icon="edit" :label="t('vitrine.calendario.editar')" class="cal__detail-btn" @click="demo('editar')" />
            <q-btn outline no-caps color="primary" icon="content_copy" :label="t('vitrine.calendario.duplicar')" class="cal__detail-btn" @click="demo('duplicar')" />
            <q-btn v-if="selected.statusEfetivo === 'EM_ANDAMENTO'" unelevated no-caps color="primary" icon="stop_circle"
              :label="t('vitrine.calendario.encerrar')" class="cal__detail-btn" @click="demo('encerrar')" />
          </div>
        </div>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup>
/*
  CALENDÁRIO / AGENDA DE RECURSOS — arquétipo de vitrine.
  Três visões sobre a MESMA fonte de eventos: Semana (grid 7 col), Mês (grid 6 semanas),
  Timeline (1 dia, uma faixa por recurso com linha do "agora"). Aritmética de data sem lib;
  cor do card = recurso (identidade), faixa lateral = status; "bloqueio" (manutenção) entra
  na mesma grade com hachura. Popover de detalhe, filtros client-side, responsivo.

  100% mock: os eventos vêm de calendario.json com offsets relativos a hoje (montados no load),
  então a demo está sempre populada e sempre tem um evento acontecendo agora. "Recurso" é
  genérico — aqui são salas, mas serve pra pessoas/veículos/equipamentos.

  Ações de escrita são simuladas (toggle canWrite + toasts demo): na tela real, trocar por
  permissão de verdade + navegação/mutação. Tokens --state-* viajam scoped-local no <style>.
*/
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue';
import { useI18n } from 'vue-i18n';
import { useQuasar } from 'quasar';
import { useNotify } from 'src/composables/useNotify';
import PageHeader from 'src/components/shared/PageHeader.vue';
import EmptyState from 'src/components/shared/EmptyState.vue';
import agendaData from 'src/vitrine/mocks/calendario.json';

const $q = useQuasar();
const { t, locale } = useI18n();
const notify = useNotify();

// ---- Permissão SIMULADA (na tela real vem de um composable de acesso/authStore) ----
const canWrite = ref(true);
// Ações que na tela real navegariam/mutariam. Aqui só um toast, pra vitrine ficar sem backend.
function demo(acao) {
  detailOpen.value = false;
  notify.info(t(`vitrine.calendario.demo_${acao}`));
}

// ---- Helpers de data (sem lib) ----
const MS_H = 3.6e6;
const startOfDay = (d) => { const x = new Date(d); x.setHours(0, 0, 0, 0); return x; };
const addDays = (d, n) => { const x = new Date(d); x.setDate(x.getDate() + n); return x; };
const startOfWeek = (d) => { const x = startOfDay(d); return addDays(x, -x.getDay()); }; // domingo → sábado
const sameDay = (a, b) => a.getFullYear() === b.getFullYear() && a.getMonth() === b.getMonth() && a.getDate() === b.getDate();
const cap = (s) => s.charAt(0).toUpperCase() + s.slice(1);
const dowShort = (d) => d.toLocaleDateString(locale.value, { weekday: 'short' }).replace(/\./g, '').toUpperCase();
const fmtTime = (d) => d.toLocaleTimeString(locale.value, { hour: '2-digit', minute: '2-digit', hour12: false });
const fmtDate = (d) => d.toLocaleDateString(locale.value, { day: '2-digit', month: '2-digit', year: 'numeric' });

// ---- Dado mock: recursos estáticos + eventos por offset, montados sobre o relógio ----
// (JSON não tem datas absolutas — um calendário com data fixa apareceria vazio fora da semana
//  da gravação; então cada evento traz diaOffset + inicio/fim "HH:MM", ou ancora:"agora".)
const MEU_ID = agendaData.usuarioAtual;
const usuariosById = Object.fromEntries((agendaData.usuarios ?? []).map((u) => [u.id, u]));
function hm(str, base) { const [h, m] = String(str).split(':').map(Number); const d = new Date(base); d.setHours(h, m || 0, 0, 0); return d; }
function materialize(base) {
  return (agendaData.eventos ?? []).map((e) => {
    let inicio, fim;
    if (e.ancora === 'agora') {
      inicio = new Date(base); inicio.setMinutes(base.getMinutes() - 45, 0, 0);
      fim = new Date(base); fim.setMinutes(base.getMinutes() + 45, 0, 0);
    } else {
      const dia = addDays(base, e.diaOffset || 0);
      inicio = hm(e.inicio, dia); fim = hm(e.fim, dia);
    }
    return {
      id: e.id, tipo: e.tipo, recursoId: e.recursoId, titulo: e.titulo,
      status: e.status || 'AGENDADA', inicio, fim,
      responsavel: e.responsavelId ? usuariosById[e.responsavelId] : null,
    };
  });
}

// ---- Cor do recurso (identidade). Genérico — não é de domínio. ----
const PALETTE = ['#3B82C4', '#4C9F70', '#E0A93B', '#C4574F', '#8E5BC4', '#2FA3A3', '#D9793E', '#5A79C4', '#B5568F', '#6FA83C'];
const HEX_RE = /^#([0-9a-fA-F]{3}|[0-9a-fA-F]{6}|[0-9a-fA-F]{8})$/;
function hashKey(key) { let h = 5381; for (let i = 0; i < key.length; i++) h = ((h * 33) ^ key.charCodeAt(i)) >>> 0; return h; }
function corRecurso(r) {
  if (r && typeof r === 'object' && HEX_RE.test(r.cor || '')) return r.cor;
  const key = (typeof r === 'string' ? r : (r?.id || r?.nome || '')) || 'recurso';
  return PALETTE[hashKey(key) % PALETTE.length];
}
const corSoft = (hex) => `color-mix(in srgb, ${hex} 16%, transparent)`;

// ---- Status efetivo do evento (derivado do relógio) → estado visual ----
// antes da janela = agendada · durante = em andamento · depois = realizada · CANCELADA persiste.
function statusEfetivo(ev, ref_) {
  if (ev.status === 'CANCELADA') return 'CANCELADA';
  if (ref_.getTime() < ev.inicio.getTime()) return 'AGENDADA';
  if (ref_.getTime() <= ev.fim.getTime()) return 'EM_ANDAMENTO';
  return 'REALIZADA';
}
const STATUS_STATE = { AGENDADA: 'scheduled', EM_ANDAMENTO: 'occupied', REALIZADA: 'ended', CANCELADA: 'blocked' };
function stateOf(ev) {
  if (ev.tipo === 'bloqueio') return 'maintenance';
  return STATUS_STATE[statusEfetivo(ev, now.value)] ?? 'scheduled';
}
function stateLabelOf(ev) {
  if (ev.tipo === 'bloqueio') return t('vitrine.calendario.bloqueio');
  const map = { AGENDADA: t('vitrine.calendario.stAgendada'), EM_ANDAMENTO: t('vitrine.calendario.stEmAndamento'), REALIZADA: t('vitrine.calendario.stRealizada') };
  return map[statusEfetivo(ev, now.value)] ?? '';
}
const dayStateLabel = (s) => ({
  available: t('vitrine.calendario.stLivre'),
  scheduled: t('vitrine.calendario.stAgendada'),
  occupied: t('vitrine.calendario.stEmAndamento'),
  maintenance: t('vitrine.calendario.bloqueio'),
}[s] ?? '');

// Cor do card = recurso; faixa lateral (box-shadow inset) = status. Bloqueio mantém hachura.
function cardStyle(ev) {
  const st = stateOf(ev);
  if (ev.tipo === 'bloqueio') return { background: `var(--state-${st}-soft)`, boxShadow: `inset 3px 0 0 var(--state-${st})` };
  const cor = corRecurso(recursosById.value[ev.recursoId]);
  return { background: corSoft(cor), boxShadow: `inset 3px 0 0 var(--state-${st})` };
}
function cardTimeColor(ev) {
  if (ev.tipo === 'bloqueio') return `var(--state-${stateOf(ev)}-text)`;
  return corRecurso(recursosById.value[ev.recursoId]);
}
function cardStatusClass(ev) {
  if (ev.tipo === 'bloqueio') return '';
  const s = statusEfetivo(ev, now.value);
  return s === 'REALIZADA' ? 'is-ended' : s === 'EM_ANDAMENTO' ? 'is-now' : '';
}
const badgeStyle = (st) => ({ background: `var(--state-${st}-soft)`, color: `var(--state-${st}-text)` });

const legend = computed(() => [
  { state: 'scheduled', label: t('vitrine.calendario.stAgendada') },
  { state: 'occupied', label: t('vitrine.calendario.stEmAndamento') },
  { state: 'ended', label: t('vitrine.calendario.stRealizada') },
  { state: 'maintenance', label: t('vitrine.calendario.bloqueio') },
]);

// ---- Estado da página ----
const view = ref($q.screen.lt.sm ? 'timeline' : 'semana');
const anchor = ref(startOfDay(new Date()));
const recursoIdFilter = ref(null);
const capacidadeFilter = ref(null);
const statusFilter = ref(null);
const minhas = ref(false);
const isMobile = computed(() => $q.screen.lt.sm);
const filtrosAbertos = ref(false);
const filtrosAtivos = computed(() =>
  (recursoIdFilter.value ? 1 : 0) + (capacidadeFilter.value ? 1 : 0) + (statusFilter.value ? 1 : 0) + (minhas.value ? 1 : 0));
function limparFiltros() { recursoIdFilter.value = null; capacidadeFilter.value = null; statusFilter.value = null; minhas.value = false; }

const recursos = ref([]);
const eventos = ref([]);
const loading = ref(false);
const carregouUmaVez = ref(false);
const atualizadoEm = ref(null);

const now = ref(new Date());
let nowTimer = null;

const viewOpts = computed(() => [
  { label: t('vitrine.calendario.viewMes'), value: 'mes' },
  { label: t('vitrine.calendario.viewSemana'), value: 'semana' },
  { label: t('vitrine.calendario.viewTimeline'), value: 'timeline' },
]);

// ---- Range do fetch acompanha a visão ----
const range = computed(() => {
  const a = startOfDay(anchor.value);
  if (view.value === 'mes') {
    const gridStart = startOfWeek(new Date(a.getFullYear(), a.getMonth(), 1));
    return { de: gridStart, ate: addDays(gridStart, 42) };
  }
  if (view.value === 'timeline') return { de: a, ate: addDays(a, 1) };
  const ini = startOfWeek(a);
  return { de: ini, ate: addDays(ini, 7) };
});

// "Feed" mock: monta os eventos sobre o relógio e fatia pela janela da visão (overlap semiaberto).
let lastKey = '';
async function fetchFeed(force = false) {
  const { de, ate } = range.value;
  const key = `${de.getTime()}|${ate.getTime()}`;
  if (!force && key === lastKey && carregouUmaVez.value) return;
  lastKey = key;
  loading.value = true;
  await Promise.resolve(); // microtask → exercita o skeleton/loading
  const todos = materialize(new Date());
  recursos.value = agendaData.recursos ?? [];
  eventos.value = todos.filter((e) => e.inicio < ate && e.fim > de);
  atualizadoEm.value = new Date();
  carregouUmaVez.value = true;
  loading.value = false;
}
watch([view, anchor], () => fetchFeed());

// ---- Navegação de período ----
function navegar(dir) {
  const d = new Date(anchor.value);
  if (view.value === 'mes') { d.setDate(1); d.setMonth(d.getMonth() + dir); }
  else if (view.value === 'timeline') d.setDate(d.getDate() + dir);
  else d.setDate(d.getDate() + 7 * dir);
  anchor.value = startOfDay(d);
}
function irHoje() { anchor.value = startOfDay(new Date()); }
function verDia(date) { anchor.value = startOfDay(date); view.value = 'timeline'; }

const dateLabel = computed(() => {
  const loc = locale.value; const a = anchor.value;
  if (view.value === 'mes') return cap(a.toLocaleDateString(loc, { month: 'long', year: 'numeric' }));
  if (view.value === 'timeline') return cap(a.toLocaleDateString(loc, { weekday: 'long', day: 'numeric', month: 'long', year: 'numeric' }));
  const ini = startOfWeek(a); const fim = addDays(ini, 6);
  try { return new Intl.DateTimeFormat(loc, { day: 'numeric', month: 'long', year: 'numeric' }).formatRange(ini, fim); }
  catch { return `${fmtDate(ini)} – ${fmtDate(fim)}`; }
});

// ---- Fatia por dia (overlap semiaberto; horários clampados ao dia) ----
const recursosById = computed(() => Object.fromEntries(recursos.value.map((s) => [s.id, s])));
function recursoNome(id) {
  const s = recursosById.value[id];
  if (!s) return '—';
  return s.unidade?.nome ? `${s.nome} · ${s.unidade.nome}` : s.nome;
}
const recursosFiltrados = computed(() => recursos.value.filter((s) => {
  if (recursoIdFilter.value && s.id !== recursoIdFilter.value) return false;
  if (capacidadeFilter.value && (s.capacidade ?? 0) < capacidadeFilter.value) return false;
  return true;
}));
const eventosFiltrados = computed(() => eventos.value.filter((e) => {
  if (recursoIdFilter.value && e.recursoId !== recursoIdFilter.value) return false;
  if (capacidadeFilter.value) {
    const s = recursosById.value[e.recursoId];
    if (!s || (s.capacidade ?? 0) < capacidadeFilter.value) return false;
  }
  if (minhas.value) { if (e.tipo !== 'evento' || e.responsavel?.id !== MEU_ID) return false; }
  if (statusFilter.value) {
    if (statusFilter.value === 'bloqueio') return e.tipo === 'bloqueio';
    if (e.tipo !== 'evento') return false;
    return statusEfetivo(e, now.value) === statusFilter.value;
  }
  return true;
}));
function eventosDoDia(dayStart) {
  const dayEnd = addDays(dayStart, 1);
  return eventosFiltrados.value
    .filter((e) => e.inicio < dayEnd && e.fim > dayStart)
    .map((e) => {
      const ini = e.inicio > dayStart ? e.inicio : dayStart;
      const fim = e.fim < dayEnd ? e.fim : dayEnd;
      return { ev: e, inicio: ini, fim, hInicio: fmtTime(ini), hFim: fim.getTime() === dayEnd.getTime() ? '23:59' : fmtTime(fim) };
    })
    .sort((x, y) => x.inicio - y.inicio || x.fim - y.fim);
}

// ---- Visão Semana ----
const week = computed(() => {
  const ini = startOfWeek(anchor.value);
  return Array.from({ length: 7 }, (_, i) => {
    const d = addDays(ini, i);
    return { date: d, wd: dowShort(d), day: d.getDate(), today: sameDay(d, now.value), events: eventosDoDia(d) };
  });
});
function novaNoDia() {
  if (!canWrite.value) return;
  demo('novo');
}

// ---- Visão Mês ----
const monthDows = computed(() => {
  const ini = startOfWeek(new Date());
  return Array.from({ length: 7 }, (_, i) => dowShort(addDays(ini, i)));
});
const monthWeeks = computed(() => {
  const a = anchor.value;
  const gridStart = startOfWeek(new Date(a.getFullYear(), a.getMonth(), 1));
  const weeks = [];
  for (let w = 0; w < 6; w++) {
    const row = [];
    for (let di = 0; di < 7; di++) {
      const dt = addDays(gridStart, w * 7 + di);
      row.push({ date: dt, day: dt.getDate(), inMonth: dt.getMonth() === a.getMonth(), today: sameDay(dt, now.value), weekend: di === 0 || di === 6, events: eventosDoDia(dt) });
    }
    weeks.push(row);
  }
  return weeks;
});

// ---- Visão Timeline (janela 07–20h) ----
const HR0 = 7, HR1 = 20, HSPAN = HR1 - HR0;
const hoursTl = Array.from({ length: HR1 - HR0 + 1 }, (_, i) => HR0 + i);
const pct = (h) => `${((h - HR0) / HSPAN) * 100}%`;
const tlRooms = computed(() => {
  const dayStart = startOfDay(anchor.value);
  const doDia = eventosDoDia(dayStart);
  return recursosFiltrados.value.map((s) => {
    const blocks = doDia
      .filter((x) => x.ev.recursoId === s.id)
      .map((x) => {
        const b0 = Math.max((x.inicio - dayStart) / MS_H, HR0);
        const b1 = Math.min((x.fim - dayStart) / MS_H, HR1);
        return b1 - b0 > 0 ? { ...x, b0, b1 } : null;
      })
      .filter(Boolean);
    return { ...s, dayState: recursoDayState(blocks), blocks };
  });
});
function recursoDayState(blocks) {
  if (blocks.some((b) => b.ev.tipo === 'bloqueio')) return 'maintenance';
  const agora = now.value;
  if (blocks.some((b) => b.ev.tipo === 'evento' && b.ev.inicio <= agora && b.ev.fim > agora)) return 'occupied';
  return blocks.length ? 'scheduled' : 'available';
}
function tlBlockStyle(b) {
  return { left: pct(b.b0), width: `${((b.b1 - b.b0) / HSPAN) * 100}%`, ...cardStyle(b.ev) };
}
const nowH = computed(() => (now.value - startOfDay(now.value)) / MS_H);
const nowVisible = computed(() => sameDay(now.value, anchor.value) && nowH.value >= HR0 && nowH.value <= HR1);
const nowLeft = computed(() => pct(nowH.value));
function onTrackClick(recurso, e) {
  if (!canWrite.value) return;
  if (e.target.closest('.cal__tl-block2')) return;
  demo('novo');
}

// ---- Cards laterais ----
const hojeInicio = computed(() => startOfDay(now.value));
const hojeNoRange = computed(() => { const { de, ate } = range.value; return hojeInicio.value >= de && hojeInicio.value < ate; });
const agendaDoDia = computed(() => (hojeNoRange.value ? eventosDoDia(hojeInicio.value) : []));
const agendaDoDiaLabel = computed(() => cap(now.value.toLocaleDateString(locale.value, { weekday: 'long', day: 'numeric', month: 'long' })));
const recursosDestaque = computed(() => {
  const doDia = agendaDoDia.value;
  return recursosFiltrados.value
    .map((s) => {
      const evs = doDia.filter((x) => x.ev.recursoId === s.id);
      return { id: s.id, nome: s.nome, unidade: s.unidade, capacidade: s.capacidade, cor: corRecurso(s), count: evs.length, dayState: recursoDayState(evs) };
    })
    .sort((a, b) => b.count - a.count || a.nome.localeCompare(b.nome));
});

// ---- Options dos filtros ----
const recursoFilterOptions = computed(() => [
  { label: t('vitrine.calendario.todos'), value: null },
  ...recursos.value.map((s) => ({ label: s.nome, value: s.id })).sort((a, b) => a.label.localeCompare(b.label)),
]);
const capacidadeOptions = computed(() => [
  { label: t('vitrine.calendario.qualquer'), value: null },
  { label: '4+', value: 4 }, { label: '8+', value: 8 }, { label: '12+', value: 12 },
]);
const statusFilterOptions = computed(() => [
  { label: t('vitrine.calendario.todos'), value: null },
  { label: t('vitrine.calendario.stAgendada'), value: 'AGENDADA' },
  { label: t('vitrine.calendario.stEmAndamento'), value: 'EM_ANDAMENTO' },
  { label: t('vitrine.calendario.stRealizada'), value: 'REALIZADA' },
  { label: t('vitrine.calendario.bloqueio'), value: 'bloqueio' },
]);

// ---- Popover de detalhe ----
const detailOpen = ref(false);
const selected = ref(null);
function openEvent(ev) {
  const multiDia = !sameDay(ev.inicio, ev.fim);
  selected.value = {
    tipo: ev.tipo,
    id: ev.id,
    titulo: ev.titulo,
    state: stateOf(ev),
    stateLabel: stateLabelOf(ev),
    recurso: recursoNome(ev.recursoId),
    dataLabel: cap(ev.inicio.toLocaleDateString(locale.value, { weekday: 'long', day: 'numeric', month: 'long', year: 'numeric' })) + (multiDia ? ` – ${fmtDate(ev.fim)}` : ''),
    horario: `${fmtTime(ev.inicio)} – ${fmtTime(ev.fim)}`,
    responsavel: ev.responsavel?.nome || '',
    statusEfetivo: ev.tipo === 'evento' ? statusEfetivo(ev, now.value) : null,
  };
  detailOpen.value = true;
}

// ---- Rodapé ----
const totalLabel = computed(() => {
  const n = eventosFiltrados.value.filter((e) => e.tipo === 'evento').length;
  return n === 1 ? `1 ${t('vitrine.calendario.eventoNoPeriodo')}` : `${n} ${t('vitrine.calendario.eventosNoPeriodo')}`;
});
const atualizadoLabel = computed(() => atualizadoEm.value ? `${t('vitrine.calendario.atualizadoAs')} ${fmtTime(atualizadoEm.value)}` : '');

onMounted(() => {
  fetchFeed();
  nowTimer = setInterval(() => { now.value = new Date(); }, 60000);
});
onBeforeUnmount(() => { if (nowTimer) clearInterval(nowTimer); });
</script>

<style scoped>
/* ============================================================================
   TOKENS --state-* — o golden tem os base (surface, border, text, color-primary),
   mas NÃO tem estes. Enviados scoped-local aqui pra tela ser drop-in sem tocar o CSS global.
   Se o mantenedor quiser promover pro design system: mover pra src/css/app.scss
   (light em :root, dark em .body--dark-app) e apagar daqui.
   ========================================================================= */
.cal {
  --state-available: #008143;   --state-available-soft: #E6F2EC;   --state-available-text: #005321;
  --state-scheduled: #0D9488;   --state-scheduled-soft: #E6F6F4;   --state-scheduled-text: #0A655D;
  --state-occupied: #F59E0B;    --state-occupied-soft: #FEF4E5;    --state-occupied-text: #915006;
  --state-ended: #878787;       --state-ended-soft: #F0F1F0;       --state-ended-text: #575756;
  --state-blocked: #DC2626;     --state-blocked-soft: #FDECEC;     --state-blocked-text: #991B1B;
  --state-maintenance: #B45309; --state-maintenance-soft: #F7EDDD; --state-maintenance-text: #8A3D0B;
}
.body--dark-app .cal {
  --state-available: #16A862;   --state-available-soft: rgba(22,168,98,0.16);    --state-available-text: #6FDCA6;
  --state-scheduled: #2CB6A8;   --state-scheduled-soft: rgba(44,182,168,0.16);   --state-scheduled-text: #74D6CB;
  --state-occupied: #F5A623;    --state-occupied-soft: rgba(245,166,35,0.16);    --state-occupied-text: #F6C878;
  --state-ended: #7C8A81;       --state-ended-soft: rgba(255,255,255,0.06);      --state-ended-text: #AEBBB2;
  --state-blocked: #F0555A;     --state-blocked-soft: rgba(240,85,90,0.16);      --state-blocked-text: #F79A9D;
  --state-maintenance: #C97A2C; --state-maintenance-soft: rgba(201,122,44,0.18); --state-maintenance-text: #E0A45F;
}

/* Layout */
.cal { background: var(--bg); }
.cal__grid { display: grid; grid-template-columns: minmax(0, 1fr) 340px; gap: 20px; align-items: start; }
.cal__main { display: flex; flex-direction: column; gap: 16px; min-width: 0; }
.cal__side { display: flex; flex-direction: column; gap: 16px; position: sticky; top: 16px; }
.cal__card { border-radius: 12px; background: var(--surface); }
.cal__pad { padding: 16px; }
.cal__calcard { position: relative; }
@media (max-width: 1180px) { .cal__grid { grid-template-columns: 1fr; } .cal__side { position: static; } }
.cal__skel { padding: 16px; border-top: 1px solid var(--border-subtle); }

/* Filtros */
.cal__filters { display: flex; gap: 12px; align-items: center; flex-wrap: wrap; padding: 16px; }
.cal__filter { flex: 1 1 130px; }
.cal__ftoggle { align-self: center; }

/* Header calendário */
.cal__calhead { display: flex; align-items: center; gap: 16px; padding: 14px 16px 0; }
.cal__seg { border: 1px solid var(--border); border-radius: 8px; }
.cal__navrow { display: flex; align-items: center; gap: 12px; padding: 14px 16px; flex-wrap: wrap; }
.cal__datelabel { font-size: 15px; font-weight: 600; color: var(--text-primary); }
.cal__legend { margin-left: auto; display: flex; gap: 14px; flex-wrap: wrap; }
.cal__legdot { display: inline-flex; align-items: center; gap: 6px; font-size: 12px; color: var(--text-secondary); }
.cal__legdot i { width: 8px; height: 8px; border-radius: 50%; }

/* Badge de estado (inline, com os tokens) */
.cal__badge { display: inline-flex; align-items: center; gap: 6px; padding: 2px 8px; border-radius: 999px; font-size: 11.5px; font-weight: 600; white-space: nowrap; }
.cal__badge i { width: 7px; height: 7px; border-radius: 50%; flex: none; }

/* Semana */
.cal__week { display: grid; grid-template-columns: repeat(7, minmax(0, 1fr)); border-top: 1px solid var(--border-subtle); overflow-x: auto; }
.cal__daycol { border-right: 1px solid var(--border-subtle); display: flex; flex-direction: column; }
.cal__daycol:last-child { border-right: 0; }
.cal__dayhead { padding: 10px; text-align: center; border-bottom: 1px solid var(--border-subtle); display: flex; flex-direction: column; align-items: center; gap: 4px; }
.cal__dayhead.is-weekend { background: var(--border-subtle); }
.cal__wd { font-size: 11px; font-weight: 700; letter-spacing: 0.05em; color: var(--text-muted); }
.cal__wd.is-today { color: var(--color-primary); }
.cal__daynum { font-size: 15px; font-weight: 700; color: var(--text-primary); font-family: 'IBM Plex Mono', monospace; height: 28px; display: flex; align-items: center; justify-content: center; }
.cal__daynum.is-today { width: 28px; border-radius: 50%; background: var(--color-primary); color: #fff; font-size: 13px; }
.cal__daybody { padding: 8px; display: flex; flex-direction: column; gap: 6px; min-height: 380px; }
.cal__daybody.is-clickable { cursor: pointer; }
.cal__ev { display: flex; flex-direction: column; gap: 1px; width: 100%; text-align: left; border: 0; cursor: pointer; border-radius: 8px; padding: 7px 9px; font: inherit; }
.cal__ev-time { font-family: 'IBM Plex Mono', monospace; font-size: 11px; font-weight: 700; font-variant-numeric: tabular-nums; }
.cal__ev-title { font-size: 12.5px; font-weight: 600; color: var(--text-primary); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.cal__ev-room { font-size: 11px; color: var(--text-muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.cal__calfoot { display: flex; justify-content: space-between; align-items: center; padding: 12px 16px; border-top: 1px solid var(--border-subtle); font-size: 12px; color: var(--text-muted); }
.cal__updated { display: flex; align-items: center; gap: 6px; }

/* Hachura de bloqueio/manutenção */
.cal__ev.is-hatch, .cal__month-ev.is-hatch { background-image: repeating-linear-gradient(45deg, transparent, transparent 5px, rgba(0,0,0,0.045) 5px, rgba(0,0,0,0.045) 10px) !important; }

/* Mês */
.cal__month { border-top: 1px solid var(--border-subtle); }
.cal__month-dows { display: grid; grid-template-columns: repeat(7, 1fr); background: var(--border-subtle); }
.cal__month-dows span { padding: 8px 10px; font-size: 11px; font-weight: 700; letter-spacing: 0.05em; color: var(--text-muted); }
.cal__month-week { display: grid; grid-template-columns: repeat(7, 1fr); border-bottom: 1px solid var(--border-subtle); }
.cal__month-cell { min-height: 108px; padding: 6px; border-right: 1px solid var(--border-subtle); display: flex; flex-direction: column; gap: 3px; min-width: 0; }
.cal__month-cell:last-child { border-right: 0; }
.cal__month-cell.is-out { background: var(--border-subtle); }
.cal__month-daynum { font-size: 12.5px; font-weight: 600; color: var(--text-secondary); font-family: 'IBM Plex Mono', monospace; align-self: flex-start; height: 24px; min-width: 24px; padding: 0 4px; display: flex; align-items: center; justify-content: center; }
.cal__month-daynum.is-today { width: 24px; min-width: 24px; padding: 0; border-radius: 50%; background: var(--color-primary); color: #fff; }
.cal__month-ev { border: 0; cursor: pointer; text-align: left; border-radius: 6px; padding: 3px 6px; font-size: 11.5px; font-weight: 600; color: var(--text-primary); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; font-family: inherit; }
.cal__month-ev-t { font-family: 'IBM Plex Mono', monospace; font-size: 10.5px; font-weight: 700; }
.cal__month-more { border: 0; background: none; cursor: pointer; font-size: 11px; font-weight: 600; color: var(--text-muted); text-align: left; padding: 1px 4px; }

/* Timeline */
.cal__timeline { border-top: 1px solid var(--border-subtle); }
.cal__tl-scroll { overflow-x: auto; }
.cal__tl-inner { min-width: 900px; }
.cal__tl-header { display: grid; grid-template-columns: 160px 1fr; background: var(--border-subtle); border-bottom: 1px solid var(--border-subtle); }
.cal__tl-hlabel { padding: 9px 12px; font-size: 11px; font-weight: 700; letter-spacing: 0.06em; text-transform: uppercase; color: var(--text-muted); }
.cal__tl-hhours { position: relative; height: 34px; }
.cal__tl-hhours span { position: absolute; top: 0; bottom: 0; display: flex; align-items: center; font-family: 'IBM Plex Mono', monospace; font-size: 10.5px; color: var(--text-muted); padding-left: 4px; }
.cal__tl-row2 { display: grid; grid-template-columns: 160px 1fr; border-top: 1px solid var(--border-subtle); min-height: 60px; }
.cal__tl-room2 { padding: 10px 12px; display: flex; align-items: center; gap: 8px; }
.cal__tl-dot { width: 8px; height: 8px; border-radius: 50%; flex: none; }
.cal__tl-rname { font-size: 14px; font-weight: 700; color: var(--text-primary); }
.cal__tl-rsub { font-size: 11px; color: var(--text-muted); }
.cal__tl-track2 { position: relative; border-left: 1px solid var(--border-subtle); }
.cal__tl-track2.is-clickable { cursor: pointer; }
.cal__tl-gridline { position: absolute; top: 0; bottom: 0; width: 1px; background: var(--border-subtle); opacity: 0.6; pointer-events: none; }
.cal__tl-now { position: absolute; top: 0; bottom: 0; width: 2px; background: var(--color-primary); z-index: 3; pointer-events: none; }
.cal__tl-block2 { position: absolute; top: 6px; bottom: 6px; border: 0; cursor: pointer; border-radius: 8px; padding: 5px 8px; overflow: hidden; text-align: left; display: flex; flex-direction: column; gap: 1px; font-family: inherit; }
.cal__tl-block2.is-hatch { background-image: repeating-linear-gradient(45deg, transparent, transparent 5px, rgba(0,0,0,0.045) 5px, rgba(0,0,0,0.045) 10px) !important; }
.cal__tl-btime { font-family: 'IBM Plex Mono', monospace; font-size: 10.5px; font-weight: 700; font-variant-numeric: tabular-nums; }
.cal__tl-btitle { font-size: 12px; font-weight: 600; color: var(--text-primary); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

/* Popover de detalhe */
.cal__detail { width: 380px; max-width: 92vw; border-radius: 12px; overflow: hidden; }
.cal__detail-strip { height: 5px; }
.cal__detail-body { padding: 18px 20px 20px; }
.cal__detail-head { display: flex; align-items: flex-start; gap: 10px; }
.cal__detail-title { font-size: 18px; font-weight: 700; line-height: 1.2; flex: 1; margin: 0; color: var(--text-primary); }
.cal__detail-badge { margin: 8px 0 4px; }
.cal__detail-row { display: flex; gap: 10px; padding: 10px 0; border-top: 1px solid var(--border-subtle); }
.cal__detail-row .q-icon { flex: none; margin-top: 1px; color: var(--text-muted); }
.cal__dk { font-size: 11px; font-weight: 600; letter-spacing: 0.03em; text-transform: uppercase; color: var(--text-muted); margin-bottom: 2px; }
.cal__dv { font-size: 14px; color: var(--text-primary); }
.cal__mono { font-family: 'IBM Plex Mono', monospace; }
.cal__detail-actions { display: flex; flex-wrap: wrap; gap: 10px; margin-top: 18px; }
.cal__detail-btn { flex: 1 1 auto; min-width: 120px; }

/* Painel lateral */
.cal__card-title { font-size: 15px; font-weight: 700; color: var(--text-primary); }
.cal__card-sub { font-size: 13px; color: var(--text-muted); margin-top: 2px; text-transform: capitalize; }
.cal__dayagenda { margin-top: 8px; }
.cal__da-row { display: flex; gap: 10px; padding: 8px 0; width: 100%; border: 0; border-top: 1px solid var(--border-subtle); background: none; cursor: pointer; text-align: left; font: inherit; align-items: center; }
.cal__da-row:first-child { border-top: 0; }
.cal__da-row:hover { background: color-mix(in srgb, var(--border-subtle) 40%, transparent); }
.cal__da-dot { width: 8px; height: 8px; border-radius: 50%; margin-top: 5px; flex: none; align-self: flex-start; }
.cal__da-body { flex: 1; min-width: 0; }
.cal__da-time { font-family: 'IBM Plex Mono', monospace; font-size: 12px; color: var(--text-muted); }
.cal__da-title { font-size: 14px; font-weight: 600; color: var(--text-primary); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.cal__da-sub { font-size: 12px; color: var(--text-muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.cal__side-empty { padding: 14px 0 4px; font-size: 13px; color: var(--text-muted); }
.cal__featured { margin-top: 8px; display: flex; flex-direction: column; gap: 10px; }
.cal__feat-row { display: flex; align-items: center; gap: 10px; }
.cal__feat-swatch { width: 28px; height: 28px; border-radius: 8px; flex: none; border: 1px solid var(--border-subtle); }
.cal__feat-body { flex: 1; min-width: 0; }
.cal__feat-name { font-size: 14px; font-weight: 600; color: var(--text-primary); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.cal__feat-sub { font-size: 12px; color: var(--text-muted); }

/* Estado do card: encerrada esmaece; em andamento ganha anel sutil */
.cal__ev.is-ended, .cal__month-ev.is-ended, .cal__tl-block2.is-ended { opacity: 0.62; }
.cal__ev.is-now, .cal__month-ev.is-now, .cal__tl-block2.is-now { outline: 1.5px solid var(--state-occupied); outline-offset: -1.5px; }

/* Mobile */
@media (max-width: 599px) {
  .cal__filtros-bar { display: flex; align-items: center; justify-content: space-between; padding: 6px 8px; }
  .cal__filters { padding: 12px; gap: 8px; }
  .cal__filter { flex: 1 1 100%; }
  .cal__calhead { padding: 12px 12px 0; }
  .cal__navrow { padding: 12px; gap: 8px; }
  .cal__week { grid-template-columns: repeat(7, minmax(150px, 1fr)); }
  .cal__daybody { min-height: 220px; }
  .cal__month { overflow-x: auto; }
  .cal__month-dows, .cal__month-week { min-width: 700px; }
}
</style>
