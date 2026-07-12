<script setup>
import { computed } from 'vue';
import { useMessageContext } from '../../provider.js';

import MessageFormatter from 'shared/helpers/MessageFormatter.js';
import { MESSAGE_VARIANTS } from '../../constants';

const props = defineProps({
  content: {
    type: String,
    required: true,
  },
});

const { variant } = useMessageContext();

const parsed = computed(() => {
  const text = props.content || '';
  // Search for the standard signature delimiter
  const parts = text.split(/\r?\n\r?\n--\r?\n\r?/);
  if (parts.length > 1) {
    const signature = parts.pop().trim();
    const body = parts.join('\n\n').trim();
    return { body, signature };
  }

  // Alternative fallback search for '--'
  const index = text.lastIndexOf('\n--\n');
  if (index !== -1) {
    const body = text.substring(0, index).trim();
    const signature = text.substring(index + 4).trim();
    return { body, signature };
  }

  return { body: text, signature: '' };
});

const formattedBody = computed(() => {
  if (variant.value === MESSAGE_VARIANTS.ACTIVITY) {
    return parsed.value.body;
  }

  return new MessageFormatter(parsed.value.body).formattedMessage;
});
</script>

<template>
  <span v-dompurify-html="formattedBody" class="prose prose-bubble" />
</template>

<style lang="scss">
.prose-bubble {
  p {
    margin-top: 0.25rem !important;
    margin-bottom: 0.25rem !important;
    line-height: 1.4 !important;
  }

  ul,
  ol {
    margin-top: 0.25rem !important;
    margin-bottom: 0.25rem !important;
    padding-left: 1.25rem !important;
  }

  li {
    margin-top: 0.125rem !important;
    margin-bottom: 0.125rem !important;
  }
}
</style>
