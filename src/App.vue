<template>
  <div class="app-shell">
    <header class="intro">
      <h1>转盘选择器</h1>
      <p>配置你的幸运选项，点击中心按钮让转盘为你做出决定。</p>
    </header>
    <WheelPicker />
    <footer class="tips">
      <p>温馨提示：输入框允许自定义任意文本内容，可根据需要设置奖励、任务或待办事项。</p>
    </footer>
  </div>
</template>

<script setup>
import WheelPicker from './components/WheelPicker.vue';
</script>

<style scoped>
.app-shell {
  display: flex;
  flex-direction: column;
  gap: 32px;
  max-width: 960px;
  margin: 0 auto;
}

.intro {
  margin: 0;
  text-align: center;
}

.intro h1 {
  margin: 0;
  font-size: 40px;
  font-weight: 700;
  color: #1f2933;
}

.intro p {
  margin-top: 12px;
  color: #475569;
  font-size: 18px;
}

.tips {
  text-align: center;
  color: #64748b;
  font-size: 14px;
}
</style>
