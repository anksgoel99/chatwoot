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
});

const emit = defineEmits(['close', 'selectAgent']);

const store = useStore();
const query = ref('');
const isRefreshing = ref(false);

const { agentsList } = useAgentsList(false);

const filteredAgents = computed(() => {
  const q = query.value.toLowerCase().trim();
  if (!q) return agentsList.value;
  return agentsList.value.filter(agent => agent.name.toLowerCase().includes(q));
});

const refreshAgents = async () => {
  isRefreshing.value = true;
  try {
    await store.dispatch('fetchAgents');
  } catch (error) {
    // Fail silently
  } finally {
    isRefreshing.value = false;
  }
};

const selectAgent = agent => {
  emit('selectAgent', agent);
};
</script>

<template>
  <Transition name="modal-fade">
    <div v-if="show" class="fixed inset-0 z-50 flex flex-col justify-end">
      <!-- Backdrop Blur Overlay -->
      <div
        class="absolute inset-0 bg-black/40 backdrop-blur-sm transition-opacity"
        @click="emit('close')"
      />

      <!-- Bottom Sheet Dialog -->
      <div
        class="relative bg-n-background dark:bg-n-solid-2 border-t border-n-weak rounded-t-3xl shadow-2xl p-4 max-h-[85vh] flex flex-col transition-all duration-300 transform translate-y-0"
      >
        <!-- Top Pull Bar Indicator -->
        <div class="flex justify-center mb-4">
          <div class="w-10 h-1 rounded-full bg-n-alpha-3" />
        </div>

        <!-- Header Title -->
        <h3 class="text-center text-base font-semibold text-n-slate-12 mb-4">
          {{ $t('CONVERSATION.HEADER.TRANSFER_TO_AGENT') }}
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
              :placeholder="$t('CONVERSATION.HEADER.SEARCH_BY_NAME')"
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

        <!-- Agents List -->
        <div
          class="flex-grow overflow-y-auto max-h-[50vh] flex flex-col gap-2 pr-1"
        >
          <button
            v-for="agent in filteredAgents"
            :key="agent.id"
            type="button"
            class="flex items-center justify-between w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
            @click="selectAgent(agent)"
          >
            <div class="flex items-center gap-3">
              <Avatar
                :name="agent.name"
                :src="agent.thumbnail"
                :size="36"
                class="shrink-0"
              />
              <span class="text-sm font-medium text-n-slate-12">
                {{ agent.name }}
              </span>
            </div>

            <!-- Presence Indicator status dot -->
            <span
              class="w-2.5 h-2.5 rounded-full"
              :class="
                agent.availability_status === 'online'
                  ? 'bg-green-500'
                  : 'bg-n-slate-6'
              "
            />
          </button>

          <!-- Empty State -->
          <div
            v-if="!filteredAgents.length"
            class="text-center py-8 text-sm text-n-slate-11"
          >
            {{ $t('CONVERSATION.HEADER.NO_AGENTS_FOUND') }}
          </div>
        </div>
      </div>
    </div>
  </Transition>
</template>
