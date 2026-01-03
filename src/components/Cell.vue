<template>
  <div 
    class="point" 
    :class="{ 
      'point-even': index % 2 === 0, 
      'point-odd': index % 2 !== 0,
      'top-row': isTop,
      'bottom-row': !isTop,
      'selected': isSelected,
      'movable': isMovable
    }"
    @click="handleClick"
  >
    <span class="point-number">{{ index + 1 }}</span>

    <div class="checkers-container">
      <div 
        v-for="n in count" 
        :key="n" 
        class="checker"
        :class="owner === 1 ? 'white-checker' : 'black-checker'"
        :style="getCheckerStyle(n)"
      ></div>
      
      <div v-if="count > 5 && isTop" class="count-badge top-badge">{{ count }}</div>
      <div v-if="count > 5 && !isTop" class="count-badge bottom-badge">{{ count }}</div>
      
      <div v-if="isMovable && count === 0" class="move-hint"></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { defineProps, defineEmits } from 'vue';

const props = defineProps<{
  index: number;
  count: number;
  owner: 0 | 1 | 2;
  isTop: boolean;
  isSelected?: boolean;
  isMovable?: boolean;
}>();

// 親コンポーネントにイベントを通知するための設定
const emit = defineEmits<{
  (e: 'clickPoint', index: number): void
}>();

function handleClick() {
  // 「このマスのインデックス」を親(Board)に伝える
  emit('clickPoint', props.index);
}

function getCheckerStyle(n: number) {
  if (n > 5) return { display: 'none' };
  return {};
}
</script>

<style scoped>
.point {
  width: 100%;
  height: 100%;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  border-left: 1px solid #ccc;
  border-right: 1px solid #ccc;
  cursor: pointer;
}

.point-even { background-color: #e0e0e0; }
.point-odd { background-color: #8f5e5e; color: white; }

/* 選択中は背景を黄色くする */
.selected {
  background-color: #ffe082 !important; 
  box-shadow: inset 0 0 10px orange;
}

/* 移動可能なら背景を緑にする */
.movable {
  background-color: #c8e6c9 !important;
  box-shadow: inset 0 0 10px green;
}

.move-hint {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background-color: rgba(0, 255, 0, 0.3);
  margin-top: 5px;
}

.top-row .checkers-container { justify-content: flex-start; }
.bottom-row .checkers-container { justify-content: flex-end; }

.point-number {
  position: absolute;
  font-size: 0.7rem;
  opacity: 0.5;
  pointer-events: none;
}
.top-row .point-number { bottom: 2px; }
.bottom-row .point-number { top: 2px; }

.checkers-container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 5px 0;
  gap: 2px;
}

.checker {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  box-shadow: 0 2px 2px rgba(0,0,0,0.3);
  z-index: 10;
  flex-shrink: 0;
}
.white-checker { background-color: #fff; border: 1px solid #ccc; }
.black-checker { background-color: #333; border: 1px solid #000; }

.count-badge {
  position: absolute;
  background: rgba(255, 0, 0, 0.8);
  color: white;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  font-size: 0.8rem;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 20;
}
.top-badge { top: 50%; }
.bottom-badge { bottom: 50%; }
</style>