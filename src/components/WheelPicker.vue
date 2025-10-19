<template>
  <div class="wheel-picker">
    <section class="wheel-panel">
      <div class="wheel-wrapper">
        <div class="pointer">
          <span class="pointer-stem"></span>
        </div>
        <div
          class="wheel"
          :style="{
            backgroundImage: wheelGradient,
            transform: `rotate(${currentRotation}deg)`,
            transitionDuration: `${spinDuration}ms`
          }"
          @transitionend="handleTransitionEnd"
        ></div>
        <button class="spin-btn" type="button" @click="spin" :disabled="isSpinning">
          {{ isSpinning ? '旋转中...' : '开始抽取' }}
        </button>
      </div>
      <p class="result" v-if="resultLabel">
        本次结果：<strong>{{ resultLabel }}</strong>
      </p>
    </section>
    <section class="config-panel">
      <header class="panel-head">
        <h2>分片配置</h2>
        <p>设置分片数量与内容，创建属于你的专属转盘。</p>
      </header>
      <div class="field">
        <label for="segment-count">分片数量</label>
        <input
          id="segment-count"
          type="number"
          min="2"
          :max="maxSegments"
          v-model.number="segmentCount"
          @change="syncSegmentCount"
          :disabled="isSpinning"
        />
        <small>建议控制在 {{ minSegments }} ~ {{ maxSegments }} 个，保证转盘清晰易读。</small>
      </div>
      <div class="segments-editor">
        <div
          v-for="(segment, index) in segments"
          :key="segment.id"
          class="segment-row"
        >
          <span class="badge" :style="{ backgroundColor: segment.color }">{{ index + 1 }}</span>
          <input
            type="text"
            v-model.trim="segment.label"
            :placeholder="`选项 ${index + 1}`"
            :disabled="isSpinning"
          />
          <button
            class="remove-btn"
            type="button"
            @click="removeSegment(index)"
            :disabled="segments.length <= minSegments || isSpinning"
          >
            删除
          </button>
        </div>
      </div>
      <div class="actions">
        <button type="button" @click="addSegment" :disabled="segments.length >= maxSegments || isSpinning">
          添加分片
        </button>
        <button type="button" @click="shuffleColors" :disabled="isSpinning">
          刷新配色
        </button>
        <button type="button" @click="resetDefaults" :disabled="isSpinning">
          恢复默认
        </button>
      </div>
    </section>
  </div>
</template>

<script setup>
import { computed, ref } from 'vue';

const minSegments = 2;
const maxSegments = 12;

const defaultLabels = ['iPhone 17', 'iPhone 17 Pro', 'iPhone 17 Pro Max'];

const curatedPalette = [
  'hsl(0deg 85% 52%)',
  'hsl(140deg 75% 45%)',
  'hsl(220deg 80% 56%)'
];

let paletteOffset = 0;

function generateColor(index) {
  const paletteLength = curatedPalette.length;
  return curatedPalette[(index + paletteOffset) % paletteLength];
}

const createInitialSegments = (labels) => {
  return labels.map((label, index) => ({
    id: index + 1,
    label,
    color: generateColor(index)
  }));
};

const segments = ref(createInitialSegments(defaultLabels));
const segmentCount = ref(segments.value.length);
let idSeed = segments.value.length;

const isSpinning = ref(false);
const spinDuration = ref(0);
const rotation = ref(0);
const resultLabel = ref('');
const pendingResultLabel = ref('');

const wheelGradient = computed(() => {
  const total = segments.value.length;
  if (!total) {
    return 'radial-gradient(#f1f5f9, #e2e8f0)';
  }
  const angle = 360 / total;
  const stops = segments.value
    .map((segment, index) => {
      const start = (index * angle).toFixed(3);
      const end = ((index + 1) * angle).toFixed(3);
      return `${segment.color} ${start}deg ${end}deg`;
    })
    .join(', ');
  return `conic-gradient(from -90deg, ${stops})`;
});

const currentRotation = computed(() => rotation.value);

const syncSegmentCount = () => {
  const sanitized = clamp(Math.round(segmentCount.value), minSegments, maxSegments);
  if (sanitized !== segmentCount.value) {
    segmentCount.value = sanitized;
  }
  resizeSegments(sanitized);
};

const resizeSegments = (target) => {
  const current = segments.value.length;
  if (target === current) {
    return;
  }
  if (target > current) {
    for (let i = 0; i < target - current; i += 1) {
      idSeed += 1;
      const nextIndex = segments.value.length;
      segments.value.push({
        id: idSeed,
        label: `选项 ${nextIndex + 1}`,
        color: generateColor(nextIndex)
      });
    }
  } else {
    segments.value.splice(target);
  }
  applyColorPalette();
  segmentCount.value = segments.value.length;
};

const addSegment = () => {
  if (segments.value.length >= maxSegments) {
    return;
  }
  resizeSegments(segments.value.length + 1);
};

const removeSegment = (index) => {
  if (segments.value.length <= minSegments) {
    return;
  }
  segments.value.splice(index, 1);
  applyColorPalette();
  segmentCount.value = segments.value.length;
};

const shuffleColors = () => {
  paletteOffset = (paletteOffset + 3) % curatedPalette.length;
  applyColorPalette();
};

const resetDefaults = () => {
  paletteOffset = 0;
  segments.value = createInitialSegments(defaultLabels);
  segmentCount.value = segments.value.length;
  idSeed = segments.value.length;
  rotation.value = 0;
  resultLabel.value = '';
  pendingResultLabel.value = '';
  spinDuration.value = 0;
  isSpinning.value = false;
};

const spin = () => {
  if (isSpinning.value) {
    return;
  }
  if (!segments.value.some((segment) => segment.label.trim().length > 0)) {
    resultLabel.value = '请先为至少一个分片填写内容';
    return;
  }
  const total = segments.value.length;
  if (!total) {
    resultLabel.value = '请至少保留两个分片';
    return;
  }

  const duration = 4200 + Math.random() * 600;
  const extraRounds = 6 + Math.floor(Math.random() * 4);
  const anglePerSegment = 360 / total;
  const winnerIndex = Math.floor(Math.random() * total);
  const winnerSegment = segments.value[winnerIndex];
  const label = winnerSegment.label.trim() || `选项 ${winnerIndex + 1}`;
  const segmentCenter = -90 + winnerIndex * anglePerSegment + anglePerSegment / 2;
  const targetNormalized = normalizeAngle(-segmentCenter);
  const startNormalized = normalizeAngle(rotation.value);
  const deltaToTarget = normalizeAngle(targetNormalized - startNormalized);

  pendingResultLabel.value = label;
  isSpinning.value = true;
  resultLabel.value = '';
  spinDuration.value = duration;
  rotation.value = rotation.value + extraRounds * 360 + deltaToTarget;
};

const handleTransitionEnd = (event) => {
  if (event.propertyName !== 'transform') {
    return;
  }
  const total = segments.value.length;
  if (!total) {
    isSpinning.value = false;
    return;
  }
  const normalized = normalizeAngle(rotation.value);

  window.requestAnimationFrame(() => {
    spinDuration.value = 0;
    rotation.value = normalized;
    isSpinning.value = false;
  });

  if (pendingResultLabel.value) {
    resultLabel.value = pendingResultLabel.value;
  }
  pendingResultLabel.value = '';
};

const applyColorPalette = () => {
  segments.value = segments.value.map((segment, index) => ({
    ...segment,
    color: generateColor(index)
  }));
};

const normalizeAngle = (angle) => {
  const normalized = angle % 360;
  return normalized < 0 ? normalized + 360 : normalized;
};

const clamp = (value, min, max) => Math.min(Math.max(value, min), max);
</script>

<style scoped>
.wheel-picker {
  display: grid;
  gap: 32px;
  grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
  background: rgba(255, 255, 255, 0.86);
  padding: 32px;
  border-radius: 24px;
  box-shadow: 0 20px 45px rgba(15, 23, 42, 0.08);
  backdrop-filter: blur(12px);
}

.wheel-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.wheel-wrapper {
  position: relative;
  width: min(360px, 80vw);
  aspect-ratio: 1;
  display: flex;
  align-items: center;
  justify-content: center;
}

.pointer {
  position: absolute;
  top: 12px;
  left: 50%;
  transform: translateX(-50%);
  width: 16px;
  height: 30%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  pointer-events: none;
  z-index: 3;
}

.pointer::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 0;
  height: 0;
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-top: 22px solid #dc2626;
  filter: drop-shadow(0 6px 10px rgba(220, 38, 38, 0.35));
}

.pointer-stem {
  width: 4px;
  height: calc(100% - 22px);
  background: linear-gradient(180deg, #f87171 0%, #b91c1c 100%);
  border-radius: 999px;
  box-shadow: 0 6px 12px rgba(248, 113, 113, 0.36);
}

.wheel {
  position: relative;
  width: 100%;
  height: 100%;
  border-radius: 50%;
  border: 12px solid #fff;
  box-shadow: inset 0 0 20px rgba(15, 23, 42, 0.08), 0 18px 24px rgba(15, 23, 42, 0.12);
  transition-property: transform;
  transition-timing-function: cubic-bezier(0.17, 0.67, 0.3, 1.35);
}

.wheel::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 70px;
  height: 70px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  background: radial-gradient(circle at 30% 30%, #fff 0%, #e2e8f0 100%);
  box-shadow: inset 0 0 12px rgba(15, 23, 42, 0.18);
  z-index: 1;
}

.spin-btn {
  position: absolute;
  width: 140px;
  height: 140px;
  border-radius: 50%;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  border: none;
  color: #fff;
  font-size: 20px;
  font-weight: 700;
  letter-spacing: 0.04em;
  cursor: pointer;
  z-index: 2;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  box-shadow: 0 20px 32px rgba(99, 102, 241, 0.35);
}

.spin-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
  box-shadow: none;
}

.spin-btn:not(:disabled):hover {
  transform: scale(1.04);
  box-shadow: 0 28px 40px rgba(124, 58, 237, 0.35);
}

.result {
  font-size: 18px;
  color: #1f2933;
  font-weight: 500;
}

.result strong {
  color: #7c3aed;
}

.config-panel {
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding: 20px;
  border-radius: 20px;
  background: rgba(248, 250, 252, 0.9);
  border: 1px solid rgba(148, 163, 184, 0.25);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.6);
}

.panel-head h2 {
  margin: 0;
  font-size: 24px;
  color: #1e293b;
}

.panel-head p {
  margin: 8px 0 0;
  color: #475569;
  font-size: 14px;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.field label {
  font-weight: 600;
  color: #1f2937;
}

.field input {
  width: 120px;
  padding: 8px 12px;
  border-radius: 12px;
  border: 1px solid #cbd5f5;
  background: #fff;
  font-size: 16px;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.field input:focus {
  outline: none;
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.18);
}

.segments-editor {
  display: flex;
  flex-direction: column;
  gap: 12px;
  max-height: 360px;
  overflow-y: auto;
  padding-right: 6px;
}

.segment-row {
  display: grid;
  grid-template-columns: auto 1fr auto;
  gap: 12px;
  align-items: center;
}

.segment-row input {
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid #d4d8e8;
  font-size: 15px;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.segment-row input:focus {
  outline: none;
  border-color: #7c3aed;
  box-shadow: 0 0 0 3px rgba(124, 58, 237, 0.18);
}

.badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  color: #fff;
  font-size: 14px;
  font-weight: 700;
  box-shadow: 0 3px 6px rgba(15, 23, 42, 0.15);
}

.remove-btn {
  border: none;
  background: #fee2e2;
  color: #b91c1c;
  font-weight: 600;
  padding: 8px 14px;
  border-radius: 10px;
  cursor: pointer;
  transition: background-color 0.2s ease, transform 0.2s ease;
}

.remove-btn:not(:disabled):hover {
  background: #fca5a5;
  transform: translateY(-1px);
}

.remove-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.actions {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
}

.actions button {
  flex: 1;
  min-width: 110px;
  border: none;
  background: linear-gradient(135deg, #38bdf8, #6366f1);
  color: #fff;
  font-weight: 600;
  padding: 10px 16px;
  border-radius: 12px;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
  box-shadow: 0 12px 20px rgba(56, 189, 248, 0.22);
}

.actions button:nth-child(2) {
  background: linear-gradient(135deg, #f97316, #fb7185);
  box-shadow: 0 12px 20px rgba(249, 115, 22, 0.22);
}

.actions button:nth-child(3) {
  background: linear-gradient(135deg, #22c55e, #14b8a6);
  box-shadow: 0 12px 20px rgba(34, 197, 94, 0.22);
}

.actions button:disabled {
  opacity: 0.7;
  cursor: not-allowed;
  box-shadow: none;
}

.actions button:not(:disabled):hover {
  transform: translateY(-1px);
  box-shadow: 0 18px 26px rgba(99, 102, 241, 0.28);
}

@media (max-width: 768px) {
  .wheel-picker {
    padding: 24px;
  }

  .spin-btn {
    width: 120px;
    height: 120px;
    font-size: 18px;
  }
}
</style>
