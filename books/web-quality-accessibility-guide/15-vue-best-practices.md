# Vueベストプラクティス - 読みやすく保守しやすいコード

## Vueの基本原則

Vue 3のComposition APIを使用したベストプラクティスを解説します。

## Composition API

### setup()

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// ✅ 良い例: Composition API
const count = ref(0)
const doubleCount = computed(() => count.value * 2)

function increment() {
  count.value++
}
</script>

<template>
  <div>
    <p>カウント: {{ count }}</p>
    <p>2倍: {{ doubleCount }}</p>
    <button @click="increment">増やす</button>
  </div>
</template>
```

## コンポーネント設計

### Props と Emits

```vue
<script setup lang="ts">
interface Props {
  title: string
  count: number
}

interface Emits {
  (e: 'update:count', value: number): void
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

function increment() {
  emit('update:count', props.count + 1)
}
</script>

<template>
  <div>
    <h2>{{ title }}</h2>
    <button @click="increment">{{ count }}</button>
  </div>
</template>
```

## アクセシビリティ

### フォーム

```vue
<script setup lang="ts">
import { ref } from 'vue'

const name = ref('')
const email = ref('')
const errors = ref<Record<string, string>>({})
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <div>
      <label for="name">
        氏名
        <span class="text-red-600" aria-label="必須">*</span>
      </label>
      <input
        id="name"
        v-model="name"
        type="text"
        required
        :aria-invalid="!!errors.name"
        :aria-describedby="errors.name ? 'name-error' : undefined"
      />
      <p v-if="errors.name" id="name-error" class="text-red-600">
        {{ errors.name }}
      </p>
    </div>
    <button type="submit">送信</button>
  </form>
</template>
```

## まとめ

VueもReactと同様、アクセシビリティとパフォーマンスのベストプラクティスを適用します。

### チェックリスト

- Composition APIを使用
- 型定義を適切に
- アクセシビリティ対応（ラベル、ARIA）

## 次のステップ

次章では、Next.jsのベストプラクティスについて詳しく解説します。
