<script setup>
import { ref, computed, watch, onMounted, nextTick, useSlots } from 'vue';
import { useMapGetter } from 'dashboard/composables/store';
import { useWindowSize } from '@vueuse/core';

const props = defineProps({
  conversationLabels: {
    type: Array,
    required: true,
  },
});

const slots = useSlots();
const accountLabels = useMapGetter('labels/getLabels');

const { width: windowWidth } = useWindowSize();
const isMobile = computed(() => windowWidth.value < 768);

const activeLabels = computed(() => {
  return accountLabels.value.filter(({ title }) =>
    props.conversationLabels.includes(title)
  );
});

const showAllLabels = ref(false);
const showExpandLabelButton = ref(false);
const labelPosition = ref(-1);
const labelContainer = ref(null);

const computeVisibleLabelPosition = () => {
  const beforeSlot = slots.before ? 100 : 0;
  if (!labelContainer.value) {
    return;
  }

  const labels = Array.from(labelContainer.value.querySelectorAll('.label'));
  let labelOffset = 0;
  showExpandLabelButton.value = false;
  labels.forEach((label, index) => {
    labelOffset += label.offsetWidth + 8;

    if (labelOffset < labelContainer.value.clientWidth - beforeSlot) {
      labelPosition.value = index;
    } else {
      showExpandLabelButton.value = labels.length > 1;
    }
  });
};

watch(activeLabels, () => {
  nextTick(() => computeVisibleLabelPosition());
});

onMounted(() => {
  computeVisibleLabelPosition();
});

const onShowLabels = e => {
  e.stopPropagation();
  showAllLabels.value = !showAllLabels.value;
  nextTick(() => computeVisibleLabelPosition());
};
</script>

<template>
  <div ref="labelContainer" v-resize="computeVisibleLabelPosition">
    <div
      v-if="activeLabels.length || $slots.before"
      class="flex items-center flex-shrink min-w-0 gap-1.5 gap-y-1"
      :class="{ 'h-auto overflow-visible flex-row flex-wrap': showAllLabels }"
    >
      <slot name="before" />
      <template v-if="isMobile && activeLabels.length > 1">
        <woot-label
          :title="activeLabels[0].title"
          :description="activeLabels[0].description"
          :color="activeLabels[0].color"
          variant="smooth"
          class="!mb-0"
          small
        />
        <span
          class="inline-flex items-center px-1.5 py-0.5 rounded-full text-xxs font-medium bg-n-slate-3 text-n-slate-11"
        >
          +{{ activeLabels.length - 1 }}
        </span>
      </template>
      <template v-else>
        <woot-label
          v-for="(label, index) in activeLabels"
          :key="label ? label.id : index"
          :title="label.title"
          :description="label.description"
          :color="label.color"
          variant="smooth"
          class="!mb-0 max-w-[calc(100%-0.5rem)] label"
          small
          :class="{
            'invisible absolute': !showAllLabels && index > labelPosition,
          }"
        />
        <button
          v-if="showExpandLabelButton"
          :title="
            showAllLabels
              ? $t('CONVERSATION.CARD.HIDE_LABELS')
              : $t('CONVERSATION.CARD.SHOW_LABELS')
          "
          class="h-5 py-0 px-1 flex-shrink-0 mr-6 ml-0 ltr:mr-6 rtl:ml-6 rtl:mr-0 rtl:rotate-180 text-n-slate-11 border-n-strong dark:border-n-strong cursor-pointer"
          @click="onShowLabels"
        >
          <fluent-icon
            :icon="showAllLabels ? 'chevron-left' : 'chevron-right'"
            size="12"
          />
        </button>
      </template>
    </div>
  </div>
</template>
