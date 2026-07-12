<script>
export default {
  inheritAttrs: false,
};
</script>

<script setup>
import { computed, useAttrs } from 'vue';
import { messageTimestamp } from 'shared/helpers/timeHelper';

import MessageStatus from './MessageStatus.vue';
import Icon from 'next/icon/Icon.vue';
import { useInbox } from 'dashboard/composables/useInbox';
import { useMessageContext } from './provider.js';

import { MESSAGE_STATUS, MESSAGE_TYPES } from './constants';

const {
  isAFacebookInbox,
  isALineChannel,
  isAPIInbox,
  isASmsInbox,
  isATelegramChannel,
  isATwilioChannel,
  isAWebWidgetInbox,
  isAWhatsAppChannel,
  isAnEmailChannel,
  isAnInstagramChannel,
  isATiktokChannel,
} = useInbox();

const {
  status,
  isPrivate,
  createdAt,
  sourceId,
  messageType,
  contentAttributes,
  content,
} = useMessageContext();

const readableTime = computed(() =>
  messageTimestamp(createdAt.value, 'LLL d, h:mm a')
);

const showStatusIndicator = computed(() => {
  if (isPrivate.value) return false;
  // Don't show status for failed messages, we already show error message
  if (status.value === MESSAGE_STATUS.FAILED) return false;
  // Don't show status for deleted messages
  if (contentAttributes.value?.deleted) return false;

  if (messageType.value === MESSAGE_TYPES.OUTGOING) return true;
  if (messageType.value === MESSAGE_TYPES.TEMPLATE) return true;

  return false;
});

const isSent = computed(() => {
  if (!showStatusIndicator.value) return false;

  // Messages will be marked as sent for the Email channel if they have a source ID.
  if (isAnEmailChannel.value) return !!sourceId.value;

  if (
    isAWhatsAppChannel.value ||
    isATwilioChannel.value ||
    isAFacebookInbox.value ||
    isASmsInbox.value ||
    isATelegramChannel.value ||
    isAnInstagramChannel.value ||
    isATiktokChannel.value
  ) {
    return sourceId.value && status.value === MESSAGE_STATUS.SENT;
  }

  // API inbox messages use real sent/delivered/read status values from the external system.
  if (isAPIInbox.value) return status.value === MESSAGE_STATUS.SENT;

  // All messages will be mark as sent for the Line channel, as there is no source ID.
  if (isALineChannel.value) return true;

  return false;
});

const isDelivered = computed(() => {
  if (!showStatusIndicator.value) return false;

  if (
    isAWhatsAppChannel.value ||
    isATwilioChannel.value ||
    isASmsInbox.value ||
    isAFacebookInbox.value ||
    isAnInstagramChannel.value ||
    isATiktokChannel.value
  ) {
    return sourceId.value && status.value === MESSAGE_STATUS.DELIVERED;
  }
  // API inbox messages use real delivered status from the external system.
  if (isAPIInbox.value) return status.value === MESSAGE_STATUS.DELIVERED;
  // All messages marked as delivered for the web widget inbox once they are sent.
  if (isAWebWidgetInbox.value) {
    return status.value === MESSAGE_STATUS.SENT;
  }
  if (isALineChannel.value) {
    return status.value === MESSAGE_STATUS.DELIVERED;
  }

  return false;
});

const isRead = computed(() => {
  if (!showStatusIndicator.value) return false;

  if (
    isAWhatsAppChannel.value ||
    isATwilioChannel.value ||
    isAFacebookInbox.value ||
    isAnInstagramChannel.value ||
    isATiktokChannel.value
  ) {
    return sourceId.value && status.value === MESSAGE_STATUS.READ;
  }

  if (isAWebWidgetInbox.value || isAPIInbox.value) {
    return status.value === MESSAGE_STATUS.READ;
  }

  return false;
});

const statusToShow = computed(() => {
  if (isRead.value) return MESSAGE_STATUS.READ;
  if (isDelivered.value) return MESSAGE_STATUS.DELIVERED;
  if (isSent.value) return MESSAGE_STATUS.SENT;

  return MESSAGE_STATUS.PROGRESS;
});

const signature = computed(() => {
  const text = content.value || '';
  // Search for standard signature delimiter
  const parts = text.split(/\r?\n\r?\n--\r?\n\r?/);
  if (parts.length > 1) {
    return parts.pop().trim();
  }
  const index = text.lastIndexOf('\n--\n');
  if (index !== -1) {
    return text.substring(index + 4).trim();
  }
  return '';
});

const attrs = useAttrs();
const cleanClasses = computed(() => {
  const originalClass = attrs.class || '';
  if (signature.value) {
    // Remove any justify- classes to avoid conflicts
    return originalClass
      .split(' ')
      .filter(c => !c.startsWith('justify-'))
      .join(' ');
  }
  return originalClass;
});
</script>

<template>
  <div
    v-bind="attrs"
    class="text-xs flex items-center mt-2 w-full select-none"
    :class="[cleanClasses, signature ? 'justify-between' : 'gap-1.5']"
    :style="attrs.style"
  >
    <div
      v-if="signature"
      class="text-[11px] text-n-slate-11 font-normal leading-normal whitespace-nowrap overflow-hidden text-ellipsis pr-4 max-w-[65%]"
    >
      {{ signature }}
    </div>
    <div class="flex items-center gap-1.5 shrink-0">
      <div class="inline">
        <time class="inline">{{ readableTime }}</time>
      </div>
      <Icon v-if="isPrivate" icon="i-lucide-lock-keyhole" class="size-3" />
      <MessageStatus v-if="showStatusIndicator" :status="statusToShow" />
    </div>
  </div>
</template>
