<script setup>
import { ref, computed } from 'vue';
import { useAgentsList } from 'dashboard/composables/useAgentsList';
import { useStore } from 'vuex';
import Avatar from 'next/avatar/Avatar.vue';

defineProps({
  show: {
    type: Boolean,
    default: false,
  },
  selectedAssigneeType: {
    type: String,
    default: 'agent',
  },
  selectedAssigneeId: {
    type: [String, Number],
    default: 'me',
  },
});

const emit = defineEmits(['close', 'select']);

const store = useStore();
const query = ref('');
const isRefreshing = ref(false);

const { agentsList } = useAgentsList(false);

const teamsList = computed(() => store.getters['teams/getTeams'] || []);

const filteredTeams = computed(() => {
  const q = query.value.toLowerCase().trim();
  if (!q) return teamsList.value;
  return teamsList.value.filter(team => team.name.toLowerCase().includes(q));
});

const filteredAgents = computed(() => {
  const q = query.value.toLowerCase().trim();
  if (!q) return agentsList.value;
  return agentsList.value.filter(agent => agent.name.toLowerCase().includes(q));
});

const handleSelect = (type, id) => {
  emit('select', { type, id });
  emit('close');
};

const refreshAgents = async () => {
  isRefreshing.value = true;
  try {
    await store.dispatch('agents/get');
    await store.dispatch('teams/get');
  } catch (error) {
    // Ignore error
  } finally {
    isRefreshing.value = false;
  }
};
</script>

<template>
  <Transition name="modal-fade">
    <div
      v-if="show"
      class="fixed inset-0 z-[110] flex flex-col justify-end sm:justify-center sm:items-center animate-fade-in"
    >
      <!-- Backdrop Overlay -->
      <div
        class="absolute inset-0 bg-black/40 backdrop-blur-sm transition-opacity"
        @click="emit('close')"
      />

      <!-- Dialog Panel -->
      <div
        class="relative bg-n-background dark:bg-n-solid-2 border border-n-weak rounded-t-3xl sm:rounded-2xl shadow-2xl p-4 w-full sm:max-w-md max-h-[85vh] sm:max-h-[70vh] flex flex-col transition-all duration-300 transform translate-y-0"
      >
        <!-- Top Pull Bar Indicator (Mobile Only) -->
        <div class="flex justify-center mb-4 sm:hidden">
          <div class="w-10 h-1 rounded-full bg-n-alpha-3" />
        </div>

        <!-- Header Title -->
        <h3
          class="text-center sm:text-left text-base font-semibold text-n-slate-12 mb-4"
        >
          {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.TITLE') }}
        </h3>

        <!-- Search Input & Refresh -->
        <div class="flex gap-2 mb-4">
          <div
            class="flex flex-1 gap-1 items-center px-3 py-0.5 rounded-xl bg-n-alpha-1 border border-n-weak focus-within:border-n-brand/40"
          >
            <span class="w-4 h-4 i-lucide-search text-n-slate-11 shrink-0" />
            <input
              v-model="query"
              type="search"
              :placeholder="
                $t('CHAT_LIST.MOBILE.AGENT_SELECT.SEARCH_PLACEHOLDER')
              "
              class="w-full h-9 bg-transparent border-none text-sm text-n-slate-12 focus:outline-none"
            />
          </div>
          <button
            type="button"
            :disabled="isRefreshing"
            class="flex items-center justify-center w-10 h-10 rounded-xl bg-n-alpha-2 hover:bg-n-alpha-3 text-n-slate-11 shrink-0 cursor-pointer disabled:opacity-50"
            @click="refreshAgents"
          >
            <span
              class="w-4.5 h-4.5 i-lucide-rotate-cw"
              :class="{ 'animate-spin': isRefreshing }"
            />
          </button>
        </div>

        <!-- Scrollable Option List -->
        <div
          class="flex-grow overflow-y-auto flex flex-col gap-1.5 pr-1 custom-thin-scrollbar"
        >
          <!-- Section 1: Quick Options -->
          <div
            class="text-xxs font-bold text-n-slate-11 uppercase tracking-wider px-2.5 mt-2 mb-1"
          >
            {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.QUICK_OPTIONS') }}
          </div>
          <!-- Option: Mine -->
          <button
            type="button"
            class="flex items-center justify-between w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
            :class="{
              'bg-n-alpha-1 border border-n-brand/20':
                selectedAssigneeType === 'agent' && selectedAssigneeId === 'me',
            }"
            @click="handleSelect('agent', 'me')"
          >
            <div class="flex items-center gap-3">
              <div
                class="flex items-center justify-center w-9 h-9 rounded-full bg-n-solid-blue text-n-blue-11 text-sm font-bold shrink-0"
              >
                {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.MINE_INITIAL') }}
              </div>
              <div>
                <div class="text-sm font-medium text-n-slate-12">
                  {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.MINE') }}
                </div>
                <div class="text-xxs text-n-slate-11">
                  {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.MINE_SUB') }}
                </div>
              </div>
            </div>
          </button>

          <!-- Option: All Conversations -->
          <button
            type="button"
            class="flex items-center justify-between w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
            :class="{
              'bg-n-alpha-1 border border-n-brand/20':
                selectedAssigneeType === 'all',
            }"
            @click="handleSelect('all', null)"
          >
            <div class="flex items-center gap-3">
              <div
                class="flex items-center justify-center w-9 h-9 rounded-full bg-n-solid-slate text-n-slate-11 text-sm font-bold shrink-0"
              >
                {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.ALL_INITIAL') }}
              </div>
              <div>
                <div class="text-sm font-medium text-n-slate-12">
                  {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.ALL') }}
                </div>
                <div class="text-xxs text-n-slate-11">
                  {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.ALL_SUB') }}
                </div>
              </div>
            </div>
          </button>

          <!-- Section 2: Teams -->
          <template v-if="filteredTeams.length">
            <div
              class="text-xxs font-bold text-n-slate-11 uppercase tracking-wider px-2.5 mt-3 mb-1"
            >
              {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.TEAMS') }}
            </div>
            <button
              v-for="team in filteredTeams"
              :key="'team-' + team.id"
              type="button"
              class="flex items-center justify-between w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
              :class="{
                'bg-n-alpha-1 border border-n-brand/20':
                  selectedAssigneeType === 'team' &&
                  selectedAssigneeId === team.id,
              }"
              @click="handleSelect('team', team.id)"
            >
              <div class="flex items-center gap-3">
                <div
                  class="flex items-center justify-center w-9 h-9 rounded-full bg-n-solid-slate text-n-slate-12 text-sm font-bold shrink-0"
                >
                  <span class="w-4 h-4 i-lucide-users" />
                </div>
                <span class="text-sm font-medium text-n-slate-12">{{
                  team.name
                }}</span>
              </div>
            </button>
          </template>

          <!-- Section 3: Agents -->
          <template v-if="filteredAgents.length">
            <div
              class="text-xxs font-bold text-n-slate-11 uppercase tracking-wider px-2.5 mt-3 mb-1"
            >
              {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.AGENTS') }}
            </div>
            <button
              v-for="agent in filteredAgents"
              :key="'agent-' + agent.id"
              type="button"
              class="flex items-center justify-between w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
              :class="{
                'bg-n-alpha-1 border border-n-brand/20':
                  selectedAssigneeType === 'agent' &&
                  selectedAssigneeId === agent.id,
              }"
              @click="handleSelect('agent', agent.id)"
            >
              <div class="flex items-center gap-3">
                <Avatar
                  :name="agent.name"
                  :src="agent.thumbnail"
                  :size="36"
                  class="shrink-0"
                />
                <span class="text-sm font-medium text-n-slate-12">{{
                  agent.name
                }}</span>
              </div>
              <span
                class="w-2.5 h-2.5 rounded-full"
                :class="
                  agent.availability_status === 'online'
                    ? 'bg-green-500'
                    : 'bg-n-slate-6'
                "
              />
            </button>
          </template>

          <!-- Empty State -->
          <div
            v-if="!filteredAgents.length && !filteredTeams.length"
            class="text-center py-8 text-sm text-n-slate-11"
          >
            {{ $t('CHAT_LIST.MOBILE.AGENT_SELECT.NO_RESULTS') }}
          </div>
        </div>
      </div>
    </div>
  </Transition>
</template>
