<script>
import { defineAsyncComponent, ref, computed } from 'vue';

import NextSidebar from 'next/sidebar/Sidebar.vue';
import WootKeyShortcutModal from 'dashboard/components/widgets/modal/WootKeyShortcutModal.vue';
import AddAccountModal from 'dashboard/components/app/AddAccountModal.vue';
import UpgradePage from 'dashboard/routes/dashboard/upgrade/UpgradePage.vue';

import { useUISettings } from 'dashboard/composables/useUISettings';
import { useAccount } from 'dashboard/composables/useAccount';
import { useWindowSize } from '@vueuse/core';

import wootConstants from 'dashboard/constants/globals';

const CommandBar = defineAsyncComponent(
  () => import('./commands/commandbar.vue')
);

const FloatingCallWidget = defineAsyncComponent(
  () => import('dashboard/components-next/call/FloatingCallWidget.vue')
);

import CopilotLauncher from 'dashboard/components-next/copilot/CopilotLauncher.vue';
import CopilotContainer from 'dashboard/components/copilot/CopilotContainer.vue';

import MobileSidebarLauncher from 'dashboard/components-next/sidebar/MobileSidebarLauncher.vue';
import { useCallsStore } from 'dashboard/stores/calls';

export default {
  components: {
    NextSidebar,
    CommandBar,
    WootKeyShortcutModal,
    AddAccountModal,
    UpgradePage,
    CopilotLauncher,
    CopilotContainer,
    FloatingCallWidget,
    MobileSidebarLauncher,
  },
  setup() {
    const upgradePageRef = ref(null);
    const { uiSettings, updateUISettings } = useUISettings();
    const { accountId, accountScopedRoute } = useAccount();
    const { width: windowWidth } = useWindowSize();
    const callsStore = useCallsStore();

    return {
      uiSettings,
      updateUISettings,
      accountId,
      accountScopedRoute,
      upgradePageRef,
      windowWidth,
      hasActiveCall: computed(() => callsStore.hasActiveCall),
      hasIncomingCall: computed(() => callsStore.hasIncomingCall),
    };
  },
  data() {
    return {
      showAccountModal: false,
      showCreateAccountModal: false,
      showShortcutModal: false,
      isMobileSidebarOpen: false,
      isKeyboardActive: false,
    };
  },
  computed: {
    isSmallScreen() {
      return this.windowWidth < wootConstants.SMALL_SCREEN_BREAKPOINT;
    },
    showUpgradePage() {
      return this.upgradePageRef?.shouldShowUpgradePage;
    },
    bypassUpgradePage() {
      return [
        'billing_settings_index',
        'settings_inbox_list',
        'general_settings_index',
        'agent_list',
      ].includes(this.$route.name);
    },
    previouslyUsedDisplayType() {
      const {
        previously_used_conversation_display_type: conversationDisplayType,
      } = this.uiSettings;
      return conversationDisplayType;
    },
  },
  watch: {
    isSmallScreen: {
      handler() {
        const { LAYOUT_TYPES } = wootConstants;
        if (window.innerWidth <= wootConstants.SMALL_SCREEN_BREAKPOINT) {
          this.updateUISettings({
            conversation_display_type: LAYOUT_TYPES.EXPANDED,
          });
        } else {
          this.updateUISettings({
            conversation_display_type: this.previouslyUsedDisplayType,
          });
        }
      },
      immediate: true,
    },
  },
  mounted() {
    if (window.visualViewport) {
      const checkKeyboard = () => {
        this.isKeyboardActive =
          window.innerHeight - window.visualViewport.height > 100;
      };
      window.visualViewport.addEventListener('resize', checkKeyboard);
      window.visualViewport.addEventListener('scroll', checkKeyboard);
      this.dashboardViewportResizeHandler = checkKeyboard;
      checkKeyboard();
    }
  },
  unmounted() {
    if (window.visualViewport && this.dashboardViewportResizeHandler) {
      window.visualViewport.removeEventListener(
        'resize',
        this.dashboardViewportResizeHandler
      );
      window.visualViewport.removeEventListener(
        'scroll',
        this.dashboardViewportResizeHandler
      );
    }
  },
  methods: {
    toggleMobileSidebar() {
      this.isMobileSidebarOpen = !this.isMobileSidebarOpen;
    },
    closeMobileSidebar() {
      this.isMobileSidebarOpen = false;
    },
    openCreateAccountModal() {
      this.showAccountModal = false;
      this.showCreateAccountModal = true;
    },
    closeCreateAccountModal() {
      this.showCreateAccountModal = false;
    },
    toggleAccountModal() {
      this.showAccountModal = !this.showAccountModal;
    },
    toggleKeyShortcutModal() {
      this.showShortcutModal = true;
    },
    closeKeyShortcutModal() {
      this.showShortcutModal = false;
    },
  },
};
</script>

<template>
  <div class="flex flex-grow overflow-hidden text-n-slate-12">
    <NextSidebar
      :is-mobile-sidebar-open="isMobileSidebarOpen"
      @toggle-account-modal="toggleAccountModal"
      @open-key-shortcut-modal="toggleKeyShortcutModal"
      @close-key-shortcut-modal="closeKeyShortcutModal"
      @show-create-account-modal="openCreateAccountModal"
      @close-mobile-sidebar="closeMobileSidebar"
    />

    <main
      class="flex flex-1 h-full w-full min-h-0 px-0 overflow-hidden bg-n-surface-1"
      :class="{ 'pb-14': isSmallScreen && !isKeyboardActive }"
    >
      <UpgradePage
        v-show="showUpgradePage"
        ref="upgradePageRef"
        :bypass-upgrade-page="bypassUpgradePage"
      >
        <MobileSidebarLauncher
          :is-mobile-sidebar-open="isMobileSidebarOpen"
          @toggle="toggleMobileSidebar"
        />
      </UpgradePage>
      <template v-if="!showUpgradePage">
        <router-view />
        <CommandBar />
        <CopilotLauncher />
        <MobileSidebarLauncher
          :is-mobile-sidebar-open="isMobileSidebarOpen"
          @toggle="toggleMobileSidebar"
        />
        <CopilotContainer />
        <FloatingCallWidget v-if="hasActiveCall || hasIncomingCall" />
      </template>
      <AddAccountModal
        :show="showCreateAccountModal"
        @close-account-create-modal="closeCreateAccountModal"
      />
      <WootKeyShortcutModal
        v-model:show="showShortcutModal"
        @close="closeKeyShortcutModal"
        @clickaway="closeKeyShortcutModal"
      />
    </main>

    <!-- Mobile Bottom Navigation Tab Bar -->
    <div
      v-if="isSmallScreen"
      v-show="!isKeyboardActive"
      class="md:hidden fixed bottom-0 left-0 right-0 z-50 flex items-center justify-around h-14 bg-n-background/90 backdrop-blur-md border-t border-n-weak pb-safe-bottom"
    >
      <RouterLink
        :to="accountScopedRoute('home')"
        class="flex flex-col items-center justify-center flex-grow text-center text-xs h-full text-n-slate-11 hover:text-n-slate-12 transition-colors duration-150"
        active-class="!text-n-brand"
      >
        <span class="i-lucide-message-circle size-5 mb-0.5" />
        <span>{{ $t('SIDEBAR.CONVERSATIONS') }}</span>
      </RouterLink>
      <RouterLink
        :to="accountScopedRoute('contacts_dashboard_index')"
        class="flex flex-col items-center justify-center flex-grow text-center text-xs h-full text-n-slate-11 hover:text-n-slate-12 transition-colors duration-150"
        active-class="!text-n-brand"
      >
        <span class="i-lucide-contact size-5 mb-0.5" />
        <span>{{ $t('SIDEBAR.CONTACTS') }}</span>
      </RouterLink>
      <RouterLink
        :to="accountScopedRoute('general_settings_index')"
        class="flex flex-col items-center justify-center flex-grow text-center text-xs h-full text-n-slate-11 hover:text-n-slate-12 transition-colors duration-150"
        active-class="!text-n-brand"
      >
        <span class="i-lucide-bolt size-5 mb-0.5" />
        <span>{{ $t('SIDEBAR.SETTINGS') }}</span>
      </RouterLink>
    </div>
  </div>
</template>
