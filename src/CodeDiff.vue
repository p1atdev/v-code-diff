<script setup lang="ts">
import { computed, ref, watch } from 'vue-demi'
import { createSplitDiff, createUnifiedDiff } from './utils'
import UnifiedViewer from './unified/UnifiedViewer.vue'
import SplitViewer from './split/SplitViewer.vue'

import './style.scss'

interface Props {
  newString: string
  oldString: string
  language?: string
  context?: number
  diffStyle?: 'word' | 'char'
  forceInlineComparison?: boolean
  outputFormat?: 'line-by-line' | 'side-by-side'
  trim?: boolean
  noDiffLineFeed?: boolean
  maxHeight?: string
  filename?: string
  newFilename?: string
  hideHeader?: boolean
  hideStat?: boolean
  theme?: 'light' | 'dark'
  // Give a pattern to ignore matching lines eg: '(time|token)'
  ignoreMatchingLines?: string
}

const props = withDefaults(defineProps<Props>(), {
  language: 'plaintext',
  context: 10,
  diffStyle: 'word',
  forceInlineComparison: false,
  outputFormat: 'line-by-line',
  trim: false,
  noDiffLineFeed: false,
  maxHeight: undefined,
  filename: undefined,
  newFilename: undefined,
  hideHeader: false,
  hideStat: false,
  theme: 'light',
  ignoreMatchingLines: undefined,
})

const emits = defineEmits<{
  (e: 'diff', diffResult: DiffResult): void
}>()

interface DiffResult {
  stat: {
    isChanged: boolean
    addNum: number
    delNum: number
  }
}

const isUnifiedViewer = computed(() => props.outputFormat === 'line-by-line')

const oldString = computed(() => {
  let value = props.oldString || ''
  value = props.trim ? value.trim() : value
  value = props.noDiffLineFeed ? value.replace(/(\r\n)/g, '\n') : value
  return value
})
const newString = computed(() => {
  let value = props.newString || ''
  value = props.trim ? value.trim() : value
  value = props.noDiffLineFeed ? value.replace(/(\r\n)/g, '\n') : value
  return value
})

const raw = computed(() =>
  isUnifiedViewer.value
    ? createUnifiedDiff(oldString.value, newString.value, props.language, props.diffStyle, props.forceInlineComparison, props.context, props.ignoreMatchingLines)
    : createSplitDiff(oldString.value, newString.value, props.language, props.diffStyle, props.forceInlineComparison, props.context, props.ignoreMatchingLines),
)
const diffChange = ref(raw.value)
const isNotChanged = computed(() => diffChange.value.stat.additionsNum === 0 && diffChange.value.stat.deletionsNum === 0)

const currentDiffIndex = ref(-1);

const goToNextDiff = () => {
  const diffs = document.querySelectorAll('.split-side-right.blob-code-addition');
  if (currentDiffIndex.value < diffs.length - 1) {
    currentDiffIndex.value++;
    updateCurrentDiffHighlight(diffs);
  }
}

const goToPrevDiff = () => {
  const diffs = document.querySelectorAll('.split-side-right.blob-code-addition');
  if (currentDiffIndex.value > 0) {
    currentDiffIndex.value--;
    updateCurrentDiffHighlight(diffs);
  }
}

const updateCurrentDiffHighlight = (diffs: NodeListOf<Element>) => {
  diffs.forEach((diff: { classList: { remove: (arg0: string) => any } }) => diff.classList.remove('current-diff'));

  const currentDiff = diffs[currentDiffIndex.value];

  if (currentDiff) {
    currentDiff.classList.add('current-diff');
    currentDiff.scrollIntoView({ behavior: 'smooth', block: 'center' });
  }
}

watch(() => props, () => {
  diffChange.value = raw.value
  emits('diff', {
    stat: {
      isChanged: !isNotChanged.value,
      addNum: diffChange.value.stat.additionsNum,
      delNum: diffChange.value.stat.deletionsNum,
    },
  })
}, { deep: true, immediate: true })
</script>

<template>
  <div class="code-diff-view" :theme="theme" :style="{ maxHeight }">
    <div v-if="!hideHeader" class="file-header">
      <!--  line by line -->
      <div v-if="isUnifiedViewer" class="file-info">
        <span>
          <div class="info-left">{{ filename }}</div>
          <div class="info-left">{{ newFilename }}</div>
        </span>
        <span v-if="!hideStat" class="diff-stat">
          <slot name="stat" :stat="diffChange.stat">
            <span class="diff-stat-added">+{{ diffChange.stat.additionsNum }} additions</span>
            <span class="diff-stat-deleted">-{{ diffChange.stat.deletionsNum }} deletions</span>
          </slot>
        </span>
      </div>
      <!-- side by side -->
      <div v-else class="file-info">
        <span class="info-left">{{ filename }}</span>
        <span class="info-right">
          <span style="margin-left: 20px;">{{ newFilename }}</span>
          <span class="diff-commandbar">
              <button class="command-item-button" title="Next Change" @click="goToNextDiff">
                <svg width="1rem" viewBox="0 0 512 512" xmlns="http://www.w3.org/2000/svg">
                  <path d="M383.6,322.7L278.6,423c-5.8,6-13.7,9-22.4,9c-8.7,0-16.5-3-22.4-9L128.4,322.7c-12.5-11.9-12.5-31.3,0-43.2  c12.5-11.9,32.7-11.9,45.2,0l50.4,48.2v-217c0-16.9,14.3-30.6,32-30.6c17.7,0,32,13.7,32,30.6v217l50.4-48.2  c12.5-11.9,32.7-11.9,45.2,0C396.1,291.4,396.1,310.7,383.6,322.7z"/>
                </svg>
              </button>
              <button class="command-item-button" title="Previous Change" @click="goToPrevDiff">
                <svg width="1rem" viewBox="0 0 512 512" xmlns="http://www.w3.org/2000/svg">
                  <path d="M128.4,189.3L233.4,89c5.8-6,13.7-9,22.4-9c8.7,0,16.5,3,22.4,9l105.4,100.3c12.5,11.9,12.5,31.3,0,43.2  c-12.5,11.9-32.7,11.9-45.2,0L288,184.4v217c0,16.9-14.3,30.6-32,30.6c-17.7,0-32-13.7-32-30.6v-217l-50.4,48.2  c-12.5,11.9-32.7,11.9-45.2,0C115.9,220.6,115.9,201.3,128.4,189.3z"/>
                </svg>
              </button>
          </span>
          <span v-if="!hideStat" class="diff-stat">         
            <slot name="stat" :stat="diffChange.stat">             
              <span class="diff-stat-added">+{{ diffChange.stat.additionsNum }} additions</span>
              <span class="diff-stat-deleted">-{{ diffChange.stat.deletionsNum }} deletions</span>
            </slot>
          </span>
        </span>
      </div>
    </div>
    <UnifiedViewer v-if="isUnifiedViewer" :diff-change="diffChange" />
    <SplitViewer v-else :diff-change="diffChange" />
  </div>
</template>

<style lang="scss">
</style>
