<script setup>
import { ref, computed } from 'vue';
import { useStore } from 'vuex';
import Avatar from 'next/avatar/Avatar.vue';

const props = defineProps({
  show: {
    type: Boolean,
    default: false,
  },
  initialFilters: {
    type: Array,
    default: () => [],
  },
});

const emit = defineEmits(['close', 'apply']);

const store = useStore();

// Local Filter States
const selectedPriority = ref(null);
const selectedInbox = ref(null);
const selectedLabels = ref([]);
const selectedParticipation = ref('any'); // 'any', 'me', 'other', 'agent'
const selectedParticipationAgentId = ref(null);
const selectedSort = ref('last_activity_at_desc');

// Data Lists from Store
const inboxes = computed(() => store.getters['inboxes/getInboxes'] || []);
const labels = computed(() => store.getters['labels/getLabels'] || []);
const agents = computed(() => store.getters['agents/getAgents'] || []);
const currentUser = computed(() => store.getters.getCurrentUser || {});

const showParticipationAgentSelect = ref(false);
const agentSearchQuery = ref('');

const filteredAgents = computed(() => {
  const q = agentSearchQuery.value.toLowerCase().trim();
  if (!q) return agents.value;
  return agents.value.filter(agent => agent.name.toLowerCase().includes(q));
});

// Initialize filters from props
const initializeFilters = () => {
  // Reset defaults
  selectedPriority.value = null;
  selectedInbox.value = null;
  selectedLabels.value = [];
  selectedParticipation.value = 'any';
  selectedParticipationAgentId.value = null;

  props.initialFilters.forEach(f => {
    if (f.attributeKey === 'priority') {
      selectedPriority.value = f.values?.[0] || null;
    } else if (f.attributeKey === 'inbox_id') {
      selectedInbox.value = f.values?.[0] || null;
    } else if (f.attributeKey === 'labels') {
      selectedLabels.value = [...(f.values || [])];
    } else if (f.attributeKey === 'assignee_id') {
      const val = f.values?.[0];
      if (f.filterOperator === 'equal_to') {
        if (val === currentUser.value.id) {
          selectedParticipation.value = 'me';
        } else {
          selectedParticipation.value = 'agent';
          selectedParticipationAgentId.value = val;
        }
      } else if (f.filterOperator === 'not_equal_to') {
        selectedParticipation.value = 'other';
      }
    }
  });

  // Load active sort from store
  selectedSort.value =
    store.getters.getChatSortFilter || 'last_activity_at_desc';
};

// Open initialization
if (props.show) {
  initializeFilters();
}

const toggleLabel = title => {
  const index = selectedLabels.value.indexOf(title);
  if (index > -1) {
    selectedLabels.value.splice(index, 1);
  } else {
    selectedLabels.value.push(title);
  }
};

const handleSelectParticipationAgent = agentId => {
  selectedParticipationAgentId.value = agentId;
  selectedParticipation.value = 'agent';
  showParticipationAgentSelect.value = false;
};

const resetAll = () => {
  selectedPriority.value = null;
  selectedInbox.value = null;
  selectedLabels.value = [];
  selectedParticipation.value = 'any';
  selectedParticipationAgentId.value = null;
  selectedSort.value = 'last_activity_at_desc';
};

const applyFilters = () => {
  const filterArray = [];

  if (selectedPriority.value) {
    filterArray.push({
      attributeKey: 'priority',
      filterOperator: 'equal_to',
      values: [selectedPriority.value],
      queryOperator: 'and',
    });
  }

  if (selectedInbox.value) {
    filterArray.push({
      attributeKey: 'inbox_id',
      filterOperator: 'equal_to',
      values: [selectedInbox.value],
      queryOperator: 'and',
    });
  }

  if (selectedLabels.value.length) {
    filterArray.push({
      attributeKey: 'labels',
      filterOperator: 'equal_to',
      values: selectedLabels.value,
      queryOperator: 'and',
    });
  }

  // Map Participation to assignee_id Advanced Filters
  if (selectedParticipation.value === 'me') {
    filterArray.push({
      attributeKey: 'assignee_id',
      filterOperator: 'equal_to',
      values: [currentUser.value.id],
      queryOperator: 'and',
    });
  } else if (selectedParticipation.value === 'other') {
    filterArray.push({
      attributeKey: 'assignee_id',
      filterOperator: 'not_equal_to',
      values: [currentUser.value.id],
      queryOperator: 'and',
    });
  } else if (
    selectedParticipation.value === 'agent' &&
    selectedParticipationAgentId.value
  ) {
    filterArray.push({
      attributeKey: 'assignee_id',
      filterOperator: 'equal_to',
      values: [selectedParticipationAgentId.value],
      queryOperator: 'and',
    });
  }

  // Update Sorting in store
  store.dispatch('setChatSortFilter', selectedSort.value);

  emit('apply', filterArray);
  emit('close');
};

const getAgentName = id => {
  const agent = agents.value.find(a => a.id === id);
  return agent ? agent.name : 'Select Agent';
};
</script>

<template>
  <Transition name="modal-fade">
    <div
      v-if="show"
      class="fixed inset-0 z-[110] flex flex-col justify-end sm:justify-center sm:items-center"
    >
      <!-- Backdrop Overlay -->
      <div
        class="absolute inset-0 bg-black/40 backdrop-blur-sm transition-opacity"
        @click="emit('close')"
      />

      <!-- Dialog Panel -->
      <div
        class="relative bg-n-background dark:bg-n-solid-2 border border-n-weak rounded-t-3xl sm:rounded-2xl shadow-2xl p-4 w-full sm:max-w-md max-h-[85vh] flex flex-col transition-all duration-300 transform translate-y-0"
      >
        <!-- Top Pull Bar Indicator (Mobile Only) -->
        <div class="flex justify-center mb-2 sm:hidden">
          <div class="w-10 h-1 rounded-full bg-n-alpha-3" />
        </div>

        <!-- Header -->
        <div
          class="flex items-center justify-between border-b border-n-weak pb-3 mb-4"
        >
          <h3 class="text-base font-semibold text-n-slate-12">
            {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.TITLE') }}
          </h3>
          <button
            type="button"
            class="text-xs font-semibold text-n-brand hover:text-n-brand-hover cursor-pointer border-none bg-transparent"
            @click="resetAll"
          >
            {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.RESET') }}
          </button>
        </div>

        <!-- Filter Sections (Scrollable) -->
        <div
          class="flex-grow overflow-y-auto flex flex-col gap-5 pr-1 mb-4 custom-thin-scrollbar"
        >
          <!-- Priority Filter -->
          <div>
            <span
              class="text-xs font-bold text-n-slate-11 uppercase tracking-wider block mb-2"
            >
              {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.PRIORITY') }}
            </span>
            <div class="flex flex-wrap gap-2">
              <button
                v-for="p in [null, 'urgent', 'high', 'medium', 'low']"
                :key="p"
                type="button"
                class="px-3 py-1.5 rounded-lg text-xs font-medium border cursor-pointer capitalize transition-all"
                :class="
                  selectedPriority === p
                    ? 'bg-n-brand text-white border-n-brand'
                    : 'bg-n-alpha-1 text-n-slate-12 border-n-weak hover:bg-n-alpha-2'
                "
                @click="selectedPriority = p"
              >
                {{ p || 'All' }}
              </button>
            </div>
          </div>

          <!-- Inbox Filter -->
          <div>
            <span
              class="text-xs font-bold text-n-slate-11 uppercase tracking-wider block mb-2"
            >
              {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.INBOX') }}
            </span>
            <div class="flex flex-col gap-1.5">
              <button
                type="button"
                class="flex items-center justify-between w-full p-2.5 rounded-xl border text-left text-xs font-medium cursor-pointer transition-all"
                :class="
                  selectedInbox === null
                    ? 'bg-n-alpha-1 border-n-brand text-n-brand'
                    : 'bg-n-surface-1 border-n-weak text-n-slate-12'
                "
                @click="selectedInbox = null"
              >
                {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.ALL_INBOXES') }}
              </button>
              <button
                v-for="inbox in inboxes"
                :key="inbox.id"
                type="button"
                class="flex items-center justify-between w-full p-2.5 rounded-xl border text-left text-xs font-medium cursor-pointer transition-all"
                :class="
                  selectedInbox === inbox.id
                    ? 'bg-n-alpha-1 border-n-brand text-n-brand'
                    : 'bg-n-surface-1 border-n-weak text-n-slate-12'
                "
                @click="selectedInbox = inbox.id"
              >
                {{ inbox.name }}
              </button>
            </div>
          </div>

          <!-- Participation (Intervened By) Filter -->
          <div>
            <span
              class="text-xs font-bold text-n-slate-11 uppercase tracking-wider block mb-2"
            >
              {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.PARTICIPATION') }}
            </span>
            <div class="flex flex-col gap-1.5">
              <button
                v-for="mode in ['any', 'me', 'other']"
                :key="mode"
                type="button"
                class="flex items-center justify-between w-full p-2.5 rounded-xl border text-left text-xs font-medium cursor-pointer transition-all"
                :class="
                  selectedParticipation === mode
                    ? 'bg-n-alpha-1 border-n-brand text-n-brand'
                    : 'bg-n-surface-1 border-n-weak text-n-slate-12'
                "
                @click="selectedParticipation = mode"
              >
                <span v-if="mode === 'any'">
                  {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.ANY_PARTICIPATION') }}
                </span>
                <span v-else-if="mode === 'me'">
                  {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.INTERVENED_BY_ME') }}
                </span>
                <span v-else>
                  {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.INTERVENED_BY_OTHER') }}
                </span>
              </button>

              <!-- Select Agent Option -->
              <button
                type="button"
                class="flex items-center justify-between w-full p-2.5 rounded-xl border text-left text-xs font-medium cursor-pointer transition-all"
                :class="
                  selectedParticipation === 'agent'
                    ? 'bg-n-alpha-1 border-n-brand text-n-brand'
                    : 'bg-n-surface-1 border-n-weak text-n-slate-12'
                "
                @click="showParticipationAgentSelect = true"
              >
                <span>
                  {{
                    selectedParticipation === 'agent' &&
                    selectedParticipationAgentId
                      ? $t(
                          'CHAT_LIST.MOBILE.FILTER_MODAL.INTERVENED_BY_AGENT',
                          { name: getAgentName(selectedParticipationAgentId) }
                        )
                      : $t(
                          'CHAT_LIST.MOBILE.FILTER_MODAL.SELECT_SPECIFIC_AGENT'
                        )
                  }}
                </span>
                <span class="i-lucide-chevron-right size-4" />
              </button>
            </div>
          </div>

          <!-- Labels Filter -->
          <div v-if="labels.length">
            <span
              class="text-xs font-bold text-n-slate-11 uppercase tracking-wider block mb-2"
            >
              {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.LABELS') }}
            </span>
            <div class="flex flex-wrap gap-1.5 max-h-40 overflow-y-auto pr-1">
              <button
                v-for="l in labels"
                :key="l.id"
                type="button"
                class="px-2.5 py-1.5 rounded-lg text-xxs font-medium border cursor-pointer transition-all flex items-center gap-1.5"
                :style="
                  selectedLabels.includes(l.title)
                    ? {
                        backgroundColor: l.color,
                        color: '#fff',
                        borderColor: l.color,
                      }
                    : {
                        backgroundColor: 'transparent',
                        color: l.color,
                        borderColor: l.color,
                      }
                "
                @click="toggleLabel(l.title)"
              >
                <span>{{ l.title }}</span>
                <span
                  v-if="selectedLabels.includes(l.title)"
                  class="i-lucide-check size-3"
                />
              </button>
            </div>
          </div>

          <!-- Sort Order Filter -->
          <div>
            <span
              class="text-xs font-bold text-n-slate-11 uppercase tracking-wider block mb-2"
            >
              {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.SORT_BY') }}
            </span>
            <div class="flex flex-col gap-1.5">
              <button
                v-for="sortOption in [
                  { label: 'Latest Activity', value: 'last_activity_at_desc' },
                  { label: 'Oldest Activity', value: 'last_activity_at_asc' },
                  { label: 'Newest Created', value: 'created_at_desc' },
                  { label: 'Oldest Created', value: 'created_at_asc' },
                ]"
                :key="sortOption.value"
                type="button"
                class="flex items-center justify-between w-full p-2.5 rounded-xl border text-left text-xs font-medium cursor-pointer transition-all"
                :class="
                  selectedSort === sortOption.value
                    ? 'bg-n-alpha-1 border-n-brand text-n-brand'
                    : 'bg-n-surface-1 border-n-weak text-n-slate-12'
                "
                @click="selectedSort = sortOption.value"
              >
                {{ sortOption.label }}
              </button>
            </div>
          </div>
        </div>

        <!-- Apply Button -->
        <button
          type="button"
          class="w-full py-3 rounded-xl bg-n-brand hover:bg-n-brand-hover text-white text-sm font-semibold cursor-pointer border-none transition-colors"
          @click="applyFilters"
        >
          {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.APPLY') }}
        </button>
      </div>
    </div>
  </Transition>

  <!-- Nested Participation Agent Selector Modal -->
  <Transition name="modal-fade">
    <div
      v-if="showParticipationAgentSelect"
      class="fixed inset-0 z-[120] flex flex-col justify-end sm:justify-center sm:items-center"
    >
      <div
        class="absolute inset-0 bg-black/40 backdrop-blur-sm transition-opacity"
        @click="showParticipationAgentSelect = false"
      />
      <div
        class="relative bg-n-background dark:bg-n-solid-2 border border-n-weak rounded-t-3xl sm:rounded-2xl shadow-2xl p-4 w-full sm:max-w-md max-h-[70vh] flex flex-col transition-all duration-300 transform translate-y-0"
      >
        <div
          class="flex items-center justify-between border-b border-n-weak pb-3 mb-4"
        >
          <h3 class="text-base font-semibold text-n-slate-12">
            {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.SELECT_AGENT') }}
          </h3>
          <button
            type="button"
            class="text-xs font-semibold text-n-slate-11 hover:text-n-slate-12 cursor-pointer border-none bg-transparent"
            @click="showParticipationAgentSelect = false"
          >
            {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.CANCEL') }}
          </button>
        </div>

        <div class="flex gap-2 mb-4">
          <div
            class="flex flex-1 gap-1 items-center px-3 py-0.5 rounded-xl bg-n-alpha-1 border border-n-weak"
          >
            <span class="w-4 h-4 i-lucide-search text-n-slate-11 shrink-0" />
            <input
              v-model="agentSearchQuery"
              type="search"
              :placeholder="
                $t('CHAT_LIST.MOBILE.FILTER_MODAL.SEARCH_AGENT_PLACEHOLDER')
              "
              class="w-full h-9 bg-transparent border-none text-sm text-n-slate-12 focus:outline-none"
            />
          </div>
        </div>

        <div
          class="flex-grow overflow-y-auto flex flex-col gap-1.5 custom-thin-scrollbar"
        >
          <button
            v-for="agent in filteredAgents"
            :key="agent.id"
            type="button"
            class="flex items-center gap-3 w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
            @click="handleSelectParticipationAgent(agent.id)"
          >
            <Avatar
              :name="agent.name"
              :src="agent.thumbnail"
              :size="36"
              class="shrink-0"
            />
            <span class="text-sm font-medium text-n-slate-12">
              {{ agent.name }}
            </span>
          </button>
          <div
            v-if="!filteredAgents.length"
            class="text-center py-8 text-sm text-n-slate-11"
          >
            {{ $t('CHAT_LIST.MOBILE.FILTER_MODAL.NO_AGENTS') }}
          </div>
        </div>
      </div>
    </div>
  </Transition>
</template>
