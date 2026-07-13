<script setup>
import { ref, computed } from 'vue';
import { useAgentsList } from 'dashboard/composables/useAgentsList';
import { useStore } from 'vuex';
import Avatar from 'next/avatar/Avatar.vue';
import { useWindowSize } from '@vueuse/core';

defineProps({
  show: {
    type: Boolean,
    default: false,
  },
  selectedAgent: {
    type: [String, Number],
    default: null,
  },
});

const emit = defineEmits(['close', 'select']);

const store = useStore();
const query = ref('');
const isRefreshing = ref(false);

const { agentsList } = useAgentsList(false);

const { width: windowWidth } = useWindowSize();
const isMobile = computed(() => windowWidth.value < 768);

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

const handleSelect = key => {
  emit('select', key);
  emit('close');
};
</script>

<template>
  <Transition name="modal-fade">
    <div v-if="show" class="fixed inset-0 z-[100] flex flex-col justify-end sm:justify-center sm:items-center">
      <!-- Backdrop Blur Overlay -->
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
        <h3 class="text-center sm:text-left text-base font-semibold text-n-slate-12 mb-4">
          Intervened by
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
              placeholder="Search by name"
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
        <div class="flex-grow overflow-y-auto flex flex-col gap-1.5 pr-1 custom-thin-scrollbar">
          <!-- Option: Me -->
          <button
            type="button"
            class="flex items-center justify-between w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
            :class="{ 'bg-n-alpha-1 border border-n-brand/20': selectedAgent === 'me' }"
            @click="handleSelect('me')"
          >
            <div class="flex items-center gap-3">
              <div class="flex items-center justify-center w-9 h-9 rounded-full bg-n-solid-blue text-n-blue-11 text-sm font-bold shrink-0">
                M
              </div>
              <span class="text-sm font-medium text-n-slate-12">
                Intervened By Me
              </span>
            </div>
          </button>

          <!-- Option: Any -->
          <button
            type="button"
            class="flex items-center justify-between w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
            :class="{ 'bg-n-alpha-1 border border-n-brand/20': !selectedAgent }"
            @click="handleSelect(null)"
          >
            <div class="flex items-center gap-3">
              <div class="flex items-center justify-center w-9 h-9 rounded-full bg-n-solid-slate text-n-slate-11 text-sm font-bold shrink-0">
                A
              </div>
              <span class="text-sm font-medium text-n-slate-12">
                Intervened By Any
              </span>
            </div>
          </button>

          <!-- Option: Other -->
          <button
            type="button"
            class="flex items-center justify-between w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
            :class="{ 'bg-n-alpha-1 border border-n-brand/20': selectedAgent === 'other' }"
            @click="handleSelect('other')"
          >
            <div class="flex items-center gap-3">
              <div class="flex items-center justify-center w-9 h-9 rounded-full bg-n-solid-purple text-n-purple-11 text-sm font-bold shrink-0">
                O
              </div>
              <span class="text-sm font-medium text-n-slate-12">
                Intervened By Other
              </span>
            </div>
          </button>

          <!-- Individual Agents -->
          <button
            v-for="agent in filteredAgents"
            :key="agent.id"
            type="button"
            class="flex items-center justify-between w-full p-2.5 hover:bg-n-alpha-2 rounded-xl text-left cursor-pointer border-none bg-transparent"
            :class="{ 'bg-n-alpha-1 border border-n-brand/20': selectedAgent === agent.id }"
            @click="handleSelect(agent.id)"
          >
            <div class="flex items-center gap-3">
              <Avatar
                :name="agent.name"
                :src="agent.thumbnail"
                :size="36"
                class="shrink-0"
              />
              <span class="text-sm font-medium text-n-slate-12">
                Intervened By {{ agent.name }}
              </span>
            </div>

            <!-- Online Dot indicator -->
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
            No agents found
          </div>
        </div>
      </div>
    </div>
  </Transition>
</template>
