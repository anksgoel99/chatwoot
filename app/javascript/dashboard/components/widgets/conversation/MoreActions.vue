<script setup>
import { computed, onUnmounted, ref } from 'vue';
import { useToggle } from '@vueuse/core';
import { useStore } from 'vuex';
import { useAlert } from 'dashboard/composables';
import { useI18n } from 'vue-i18n';
import { emitter } from 'shared/helpers/mitt';
import EmailTranscriptModal from './EmailTranscriptModal.vue';
import ResolveAction from '../../buttons/ResolveAction.vue';
import TransferModal from './TransferModal.vue';
import ButtonV4 from 'dashboard/components-next/button/Button.vue';
import DropdownMenu from 'dashboard/components-next/dropdown-menu/DropdownMenu.vue';
import { useAgentsList } from 'dashboard/composables/useAgentsList';
import { useWindowSize } from '@vueuse/core';
import Avatar from 'next/avatar/Avatar.vue';

import {
  CMD_MUTE_CONVERSATION,
  CMD_SEND_TRANSCRIPT,
  CMD_UNMUTE_CONVERSATION,
} from 'dashboard/helper/commandbar/events';

// No props needed as we're getting currentChat from the store directly
const store = useStore();

const showTransferModal = ref(false);
const showTransferDropdown = ref(false);
const transferQuery = ref('');
const { agentsList } = useAgentsList(false);

const { width: windowWidth } = useWindowSize();
const isMobile = computed(() => windowWidth.value < 768);

const filteredAgents = computed(() => {
  const q = transferQuery.value.toLowerCase().trim();
  if (!q) return agentsList.value;
  return agentsList.value.filter(agent =>
    agent.name.toLowerCase().includes(q)
  );
});

const handleSelectAgent = agent => {
  const agentId = agent ? agent.id : null;
  store.dispatch('setCurrentChatAssignee', {
    conversationId: currentChat.value.id,
    assignee: agent,
  });
  store
    .dispatch('assignAgent', {
      conversationId: currentChat.value.id,
      agentId,
    })
    .then(() => {
      useAlert(t('CONVERSATION.CHANGE_AGENT'));
      showTransferModal.value = false;
      showTransferDropdown.value = false;
    });
};
const { t } = useI18n();

const [showEmailActionsModal, toggleEmailModal] = useToggle(false);
const [showActionsDropdown, toggleDropdown] = useToggle(false);

const currentChat = computed(() => store.getters.getSelectedChat);

const actionMenuItems = computed(() => {
  const items = [];

  if (!currentChat.value.muted) {
    items.push({
      icon: 'i-lucide-volume-off',
      label: t('CONTACT_PANEL.MUTE_CONTACT'),
      action: 'mute',
      value: 'mute',
    });
  } else {
    items.push({
      icon: 'i-lucide-volume-1',
      label: t('CONTACT_PANEL.UNMUTE_CONTACT'),
      action: 'unmute',
      value: 'unmute',
    });
  }

  items.push({
    icon: 'i-lucide-share',
    label: t('CONTACT_PANEL.SEND_TRANSCRIPT'),
    action: 'send_transcript',
    value: 'send_transcript',
  });

  return items;
});

const handleActionClick = ({ action }) => {
  toggleDropdown(false);

  if (action === 'mute') {
    store.dispatch('muteConversation', currentChat.value.id);
    useAlert(t('CONTACT_PANEL.MUTED_SUCCESS'));
  } else if (action === 'unmute') {
    store.dispatch('unmuteConversation', currentChat.value.id);
    useAlert(t('CONTACT_PANEL.UNMUTED_SUCCESS'));
  } else if (action === 'send_transcript') {
    toggleEmailModal();
  }
};

// These functions are needed for the event listeners
const mute = () => {
  store.dispatch('muteConversation', currentChat.value.id);
  useAlert(t('CONTACT_PANEL.MUTED_SUCCESS'));
};

const unmute = () => {
  store.dispatch('unmuteConversation', currentChat.value.id);
  useAlert(t('CONTACT_PANEL.UNMUTED_SUCCESS'));
};

emitter.on(CMD_MUTE_CONVERSATION, mute);
emitter.on(CMD_UNMUTE_CONVERSATION, unmute);
emitter.on(CMD_SEND_TRANSCRIPT, toggleEmailModal);

onUnmounted(() => {
  emitter.off(CMD_MUTE_CONVERSATION, mute);
  emitter.off(CMD_UNMUTE_CONVERSATION, unmute);
  emitter.off(CMD_SEND_TRANSCRIPT, toggleEmailModal);
});
</script>

<template>
  <div class="relative flex items-center gap-2 actions--container">
    <!-- Mobile: icon button opening bottom sheet modal -->
    <ButtonV4
      v-if="isMobile"
      size="sm"
      variant="ghost"
      color="slate"
      icon="i-lucide-user-plus"
      v-tooltip="t('CONVERSATION.HEADER.TRANSFER')"
      class="rounded-md hover:bg-n-alpha-2"
      @click="showTransferModal = true"
    />
    <!-- Desktop: Text dropdown opening inline popover -->
    <div v-else class="relative flex items-center">
      <ButtonV4
        size="sm"
        variant="faint"
        color="slate"
        icon="i-lucide-user-round-plus"
        trailing-icon="i-lucide-chevron-down"
        class="rounded-lg border border-n-weak font-medium text-xs py-1.5 px-3 bg-white dark:bg-n-solid-2 text-n-slate-12 hover:bg-n-alpha-1 shrink-0"
        @click="showTransferDropdown = !showTransferDropdown"
      >
        Transfer To
      </ButtonV4>
      
      <!-- Inline Popover Dropdown -->
      <div
        v-if="showTransferDropdown"
        v-on-clickaway="() => showTransferDropdown = false"
        class="absolute right-0 top-full mt-1.5 z-[100] w-64 bg-white dark:bg-n-solid-2 border border-n-weak rounded-xl shadow-xl p-3 flex flex-col gap-2 transition-all duration-150 animate-fade-in"
      >
        <!-- Search Input -->
        <div class="flex gap-1.5 items-center px-2 py-1 rounded-lg bg-n-alpha-1 border border-n-weak focus-within:border-n-brand/40">
          <span class="w-3.5 h-3.5 i-lucide-search text-n-slate-11 shrink-0" />
          <input
            v-model="transferQuery"
            type="search"
            placeholder="Search by name..."
            class="w-full h-7 bg-transparent border-none text-xs text-n-slate-12 focus:outline-none placeholder-n-slate-10"
          />
        </div>

        <!-- List of Agents -->
        <div class="max-h-60 overflow-y-auto flex flex-col gap-1 pr-0.5 custom-thin-scrollbar">
          <button
            v-for="agent in filteredAgents"
            :key="agent.id"
            type="button"
            class="flex items-center justify-between w-full p-1.5 hover:bg-n-alpha-2 rounded-lg text-left cursor-pointer border-none bg-transparent"
            @click="handleSelectAgent(agent)"
          >
            <div class="flex items-center gap-2">
              <Avatar
                :name="agent.name"
                :src="agent.thumbnail"
                :size="24"
                class="shrink-0"
              />
              <span class="text-xs font-medium text-n-slate-12">
                {{ agent.name }}
              </span>
            </div>
            <span
              class="w-2 h-2 rounded-full"
              :class="
                agent.availability_status === 'online'
                  ? 'bg-green-500'
                  : 'bg-n-slate-6'
              "
            />
          </button>
          <div
            v-if="!filteredAgents.length"
            class="text-center py-4 text-xs text-n-slate-11"
          >
            No agents found
          </div>
        </div>
      </div>
    </div>
    <ResolveAction
      :conversation-id="currentChat.id"
      :status="currentChat.status"
    />
    <div
      v-on-clickaway="() => toggleDropdown(false)"
      class="relative flex items-center group"
    >
      <ButtonV4
        v-tooltip="$t('CONVERSATION.HEADER.MORE_ACTIONS')"
        size="sm"
        variant="ghost"
        color="slate"
        icon="i-lucide-more-vertical"
        class="rounded-md group-hover:bg-n-alpha-2"
        @click="toggleDropdown()"
      />
      <DropdownMenu
        v-if="showActionsDropdown"
        :menu-items="actionMenuItems"
        class="mt-1 ltr:right-0 rtl:left-0 top-full"
        @action="handleActionClick"
      />
    </div>
    <EmailTranscriptModal
      v-if="showEmailActionsModal"
      :show="showEmailActionsModal"
      :current-chat="currentChat"
      @cancel="toggleEmailModal"
    />
    <TransferModal
      v-if="showTransferModal"
      :show="showTransferModal"
      @close="showTransferModal = false"
      @select-agent="handleSelectAgent"
    />
  </div>
</template>
