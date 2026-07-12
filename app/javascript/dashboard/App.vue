<script>
import { mapGetters } from 'vuex';
import LoadingState from './components/widgets/LoadingState.vue';
import NetworkNotification from './components/NetworkNotification.vue';
import UpdateBanner from './components/app/UpdateBanner.vue';
import StatusBanner from './components/app/StatusBanner.vue';
import PaymentPendingBanner from './components/app/PaymentPendingBanner.vue';
import PendingEmailVerificationBanner from './components/app/PendingEmailVerificationBanner.vue';
import vueActionCable from './helper/actionCable';
import { useRouter } from 'vue-router';
import { useStore } from 'dashboard/composables/store';
import WootSnackbarBox from './components/SnackbarContainer.vue';
import { setColorTheme } from './helper/themeHelper';
import { isOnOnboardingView } from 'v3/helpers/RouteHelper';
import { useAccount } from 'dashboard/composables/useAccount';
import { useFontSize } from 'dashboard/composables/useFontSize';
import {
  registerSubscription,
  verifyServiceWorkerExistence,
} from './helper/pushHelper';
import ReconnectService from 'dashboard/helper/ReconnectService';
import { useUISettings } from 'dashboard/composables/useUISettings';

export default {
  name: 'App',

  components: {
    LoadingState,
    NetworkNotification,
    UpdateBanner,
    StatusBanner,
    PaymentPendingBanner,
    WootSnackbarBox,
    PendingEmailVerificationBanner,
  },
  setup() {
    const router = useRouter();
    const store = useStore();
    const { accountId } = useAccount();
    // Use the font size composable (it automatically sets up the watcher)
    const { currentFontSize } = useFontSize();
    const { uiSettings } = useUISettings();

    return {
      router,
      store,
      currentAccountId: accountId,
      currentFontSize,
      uiSettings,
    };
  },
  data() {
    return {
      latestChatwootVersion: null,
      reconnectService: null,
      isTapifyCompactUI: localStorage.getItem('tapifyCompactUI') === 'true',
    };
  },
  computed: {
    ...mapGetters({
      getAccount: 'accounts/getAccount',
      isRTL: 'accounts/isRTL',
      currentUser: 'getCurrentUser',
      authUIFlags: 'getAuthUIFlags',
    }),
    hideOnOnboardingView() {
      return !isOnOnboardingView(this.$route);
    },
  },

  watch: {
    currentAccountId: {
      immediate: true,
      handler() {
        if (this.currentAccountId) {
          this.initializeAccount();
        }
      },
    },
  },
  mounted() {
    this.initializeColorTheme();
    this.listenToThemeChanges();
    // If user locale is set, use it; otherwise use account locale
    this.setLocale(
      this.uiSettings?.locale || window.chatwootConfig.selectedLocale
    );
  },
  unmounted() {
    if (this.reconnectService) {
      this.reconnectService.disconnect();
    }
  },
  methods: {
    toggleTapifyCompactUI() {
      this.isTapifyCompactUI = !this.isTapifyCompactUI;
      localStorage.setItem(
        'tapifyCompactUI',
        this.isTapifyCompactUI ? 'true' : 'false'
      );
    },
    initializeColorTheme() {
      setColorTheme(window.matchMedia('(prefers-color-scheme: dark)').matches);
    },
    listenToThemeChanges() {
      const mql = window.matchMedia('(prefers-color-scheme: dark)');
      mql.onchange = e => setColorTheme(e.matches);
    },
    setLocale(locale) {
      if (locale) {
        this.$root.$i18n.locale = locale;
      }
    },
    async initializeAccount() {
      await this.$store.dispatch('accounts/get');
      this.$store.dispatch('setActiveAccount', {
        accountId: this.currentAccountId,
      });
      const account = this.getAccount(this.currentAccountId);
      const { locale, latest_chatwoot_version: latestChatwootVersion } =
        account;
      const { pubsub_token: pubsubToken } = this.currentUser || {};
      // If user locale is set, use it; otherwise use account locale
      this.setLocale(this.uiSettings?.locale || locale);
      this.latestChatwootVersion = latestChatwootVersion;
      vueActionCable.init(this.store, pubsubToken);
      this.reconnectService = new ReconnectService(this.store, this.router);
      window.reconnectService = this.reconnectService;

      verifyServiceWorkerExistence(registration =>
        registration.pushManager.getSubscription().then(subscription => {
          if (subscription) {
            registerSubscription();
          }
        })
      );
    },
  },
};
</script>

<template>
  <div
    v-if="!authUIFlags.isFetching"
    id="app"
    class="flex flex-col w-full h-screen min-h-0 bg-n-background tapify-compact-ui"
    :dir="isRTL ? 'rtl' : 'ltr'"
  >
    <UpdateBanner :latest-chatwoot-version="latestChatwootVersion" />
    <StatusBanner />
    <template v-if="currentAccountId">
      <PendingEmailVerificationBanner v-if="hideOnOnboardingView" />
      <PaymentPendingBanner v-if="hideOnOnboardingView" />
    </template>
    <router-view v-slot="{ Component }">
      <transition name="fade" mode="out-in">
        <component :is="Component" />
      </transition>
    </router-view>
    <WootSnackbarBox />
    <!-- <NetworkNotification /> -->
  </div>
  <LoadingState v-else />
</template>

<style lang="scss">
@import './assets/scss/app';

.v-popper--theme-tooltip .v-popper__inner {
  background: black !important;
  font-size: 0.75rem;
  padding: 4px 8px !important;
  border-radius: 6px;
  font-weight: 400;
}

.v-popper--theme-tooltip .v-popper__arrow-container {
  display: none;
}

.tapify-compact-ui [data-bubble-name='image'] {
  padding: 6px !important;
}

.tapify-compact-ui [data-bubble-name='image'] img {
  max-width: 230px !important;
  max-height: 190px !important;
  object-fit: contain !important;
  border-radius: 8px !important;
}

.tapify-compact-ui [data-bubble-name='attachment'] {
  padding: 8px !important;
}

.tapify-compact-ui [data-bubble-name] {
  font-size: 13px !important;
  line-height: 1.35 !important;
}

.tapify-compact-ui [data-bubble-name='text'] {
  width: fit-content !important;
  max-width: min(340px, 80%) !important;
  padding: 8px 11px !important;
}

.tapify-compact-ui [data-bubble-name='text'] > div {
  gap: 4px !important;
}

.tapify-compact-ui [data-bubble-name='text'] .prose-bubble {
  font-size: 13px !important;
  line-height: 1.34 !important;
}

.tapify-compact-ui [data-bubble-name='text'] .prose-bubble p {
  margin-top: 0.6em !important;
  margin-bottom: 0.6em !important;
}

.tapify-compact-ui [data-bubble-name='text'] .prose-bubble br {
  display: block;
  content: '';
  margin-top: 0.2em;
}

.tapify-compact-ui [data-bubble-name='text'] [class*='message-meta'],
.tapify-compact-ui [data-bubble-name='text'] time {
  margin-top: 4px !important;
  font-size: 11px !important;
}

.tapify-compact-ui .message-bubble-container {
  margin-bottom: 5px !important;
}

.tapify-compact-ui textarea {
  min-height: 40px !important;
  font-size: 13px !important;
}

.tapify-compact-ui .message,
.tapify-compact-ui [class*='message'],
.tapify-compact-ui [class*='Message'] {
  margin-top: 4px !important;
  margin-bottom: 4px !important;
}

.tapify-compact-ui [class*='conversation'],
.tapify-compact-ui [class*='Conversation'] {
  font-size: 13px;
}

.tapify-compact-ui [class*='sidebar'],
.tapify-compact-ui [class*='Sidebar'] {
  font-size: 13px;
}

.tapify-compact-ui [class*='card'],
.tapify-compact-ui [class*='Card'] {
  padding-top: 8px !important;
  padding-bottom: 8px !important;
}
</style>
