# Vueベストプラクティス - 読みやすく保守しやすいコード

## Vueの基本原則

Vue 3のComposition APIは、ロジックの再利用性と型安全性を向上させます。多くのプロジェクトで採用されている実践的なパターンを解説します。

## Composition API

### setup()

`<script setup>`は、Composition APIをより簡潔に書ける構文糖衣です。

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

### リアクティブな状態管理

```vue
<script setup lang="ts">
import { ref, reactive, computed } from 'vue'

// ✅ 良い例: プリミティブ値にはref
const count = ref(0)
const message = ref('')

// ✅ 良い例: オブジェクトにはreactive
const state = reactive({
  loading: false,
  error: null as Error | null,
  data: [] as User[],
})

// ✅ 良い例: 複雑な計算にはcomputed
const filteredData = computed(() =>
  state.data.filter(user => user.active)
)

interface User {
  id: string
  name: string
  active: boolean
}
</script>
```

### watchとwatchEffect

```vue
<script setup lang="ts">
import { ref, watch, watchEffect } from 'vue'

const count = ref(0)
const userId = ref('')

// ✅ 良い例: 特定の値を監視
watch(count, (newValue, oldValue) => {
  console.log(`カウントが ${oldValue} から ${newValue} に変更されました`)
})

// ✅ 良い例: 複数の値を監視
watch([count, userId], ([newCount, newUserId]) => {
  console.log(`カウント: ${newCount}, ユーザーID: ${newUserId}`)
})

// ✅ 良い例: 自動依存関係追跡
watchEffect(() => {
  console.log(`カウント: ${count.value}`)
  // count.valueを使用しているため、自動的に監視される
})
</script>
```

## コンポーネント設計

### Props と Emits

型安全なpropsとemitsの定義方法です。

```vue
<script setup lang="ts">
interface Props {
  title: string
  count: number
  disabled?: boolean
}

interface Emits {
  (e: 'update:count', value: number): void
  (e: 'submit', data: { count: number }): void
}

// ✅ 良い例: 型安全なprops
const props = withDefaults(defineProps<Props>(), {
  disabled: false,
})

const emit = defineEmits<Emits>()

function increment() {
  emit('update:count', props.count + 1)
}

function handleSubmit() {
  emit('submit', { count: props.count })
}
</script>

<template>
  <div>
    <h2>{{ title }}</h2>
    <button
      @click="increment"
      :disabled="disabled"
    >
      {{ count }}
    </button>
    <button @click="handleSubmit">送信</button>
  </div>
</template>
```

### v-modelのカスタマイズ

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import CustomInput from './CustomInput.vue'

const searchQuery = ref('')
</script>

<template>
  <!-- ✅ 良い例: v-modelを使用 -->
  <CustomInput v-model="searchQuery" />
  <p>検索キーワード: {{ searchQuery }}</p>
</template>
```

```vue
<!-- CustomInput.vue -->
<script setup lang="ts">
interface Props {
  modelValue: string
}

interface Emits {
  (e: 'update:modelValue', value: string): void
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

function handleInput(event: Event) {
  const target = event.target as HTMLInputElement
  emit('update:modelValue', target.value)
}
</script>

<template>
  <input
    :value="modelValue"
    @input="handleInput"
    type="text"
    class="border rounded px-3 py-2"
  />
</template>
```

### Composables（再利用可能なロジック）

```typescript
// composables/useUser.ts
import { ref, computed } from 'vue'

export function useUser(userId: string) {
  const user = ref<User | null>(null)
  const loading = ref(true)
  const error = ref<Error | null>(null)

  async function fetchUser() {
    loading.value = true
    try {
      const response = await fetch(`/api/users/${userId}`)
      user.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  const displayName = computed(() => {
    return user.value ? `${user.value.firstName} ${user.value.lastName}` : ''
  })

  fetchUser()

  return {
    user,
    loading,
    error,
    displayName,
    refetch: fetchUser,
  }
}

interface User {
  id: string
  firstName: string
  lastName: string
  email: string
}
```

```vue
<!-- 使用例 -->
<script setup lang="ts">
import { useUser } from '@/composables/useUser'

const props = defineProps<{ userId: string }>()
const { user, loading, error, displayName } = useUser(props.userId)
</script>

<template>
  <div>
    <div v-if="loading">読み込み中...</div>
    <div v-else-if="error">エラー: {{ error.message }}</div>
    <div v-else-if="user">
      <h2>{{ displayName }}</h2>
      <p>{{ user.email }}</p>
    </div>
  </div>
</template>
```

## ディレクティブのベストプラクティス

### v-ifとv-show

```vue
<template>
  <!-- ✅ 良い例: 頻繁に切り替わる場合はv-show -->
  <div v-show="isVisible">
    頻繁に表示/非表示が切り替わるコンテンツ
  </div>

  <!-- ✅ 良い例: 条件が変わりにくい場合はv-if -->
  <div v-if="user.isAdmin">
    管理者専用コンテンツ
  </div>

  <!-- ✅ 良い例: v-else-if, v-else を活用 -->
  <div v-if="status === 'loading'">読み込み中...</div>
  <div v-else-if="status === 'error'">エラーが発生しました</div>
  <div v-else>コンテンツ表示</div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const isVisible = ref(true)
const status = ref<'loading' | 'error' | 'success'>('loading')
const user = ref({ isAdmin: false })
</script>
```

### v-forとkey

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Todo {
  id: string
  text: string
  completed: boolean
}

const todos = ref<Todo[]>([
  { id: '1', text: 'タスク1', completed: false },
  { id: '2', text: 'タスク2', completed: true },
])
</script>

<template>
  <!-- ✅ 良い例: 一意なkeyを使用 -->
  <ul>
    <li
      v-for="todo in todos"
      :key="todo.id"
    >
      {{ todo.text }}
    </li>
  </ul>

  <!-- ❌ 悪い例: インデックスをkeyに使用 -->
  <ul>
    <li
      v-for="(todo, index) in todos"
      :key="index"
    >
      {{ todo.text }}
    </li>
  </ul>
</template>
```

## アクセシビリティ

### フォーム

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface FormData {
  name: string
  email: string
  message: string
}

const form = ref<FormData>({
  name: '',
  email: '',
  message: '',
})

const errors = ref<Partial<Record<keyof FormData, string>>>({})

function validateForm(): boolean {
  errors.value = {}

  if (!form.value.name) {
    errors.value.name = '氏名を入力してください'
  }

  if (!form.value.email) {
    errors.value.email = 'メールアドレスを入力してください'
  } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(form.value.email)) {
    errors.value.email = '有効なメールアドレスを入力してください'
  }

  if (!form.value.message) {
    errors.value.message = 'メッセージを入力してください'
  }

  return Object.keys(errors.value).length === 0
}

function handleSubmit() {
  if (validateForm()) {
    console.log('フォーム送信:', form.value)
  }
}

const isNameInvalid = computed(() => !!errors.value.name)
const isEmailInvalid = computed(() => !!errors.value.email)
const isMessageInvalid = computed(() => !!errors.value.message)
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <!-- 氏名フィールド -->
    <div class="mb-4">
      <label for="name" class="block mb-2">
        氏名
        <span class="text-red-600" aria-label="必須">*</span>
      </label>
      <input
        id="name"
        v-model="form.name"
        type="text"
        required
        aria-required="true"
        :aria-invalid="isNameInvalid"
        :aria-describedby="errors.name ? 'name-error' : undefined"
        class="border rounded px-3 py-2 w-full"
      />
      <p
        v-if="errors.name"
        id="name-error"
        class="text-red-600 text-sm mt-1"
        role="alert"
      >
        {{ errors.name }}
      </p>
    </div>

    <!-- メールアドレスフィールド -->
    <div class="mb-4">
      <label for="email" class="block mb-2">
        メールアドレス
        <span class="text-red-600" aria-label="必須">*</span>
      </label>
      <input
        id="email"
        v-model="form.email"
        type="email"
        required
        aria-required="true"
        :aria-invalid="isEmailInvalid"
        :aria-describedby="errors.email ? 'email-error' : undefined"
        class="border rounded px-3 py-2 w-full"
      />
      <p
        v-if="errors.email"
        id="email-error"
        class="text-red-600 text-sm mt-1"
        role="alert"
      >
        {{ errors.email }}
      </p>
    </div>

    <!-- メッセージフィールド -->
    <div class="mb-4">
      <label for="message" class="block mb-2">
        メッセージ
        <span class="text-red-600" aria-label="必須">*</span>
      </label>
      <textarea
        id="message"
        v-model="form.message"
        required
        aria-required="true"
        :aria-invalid="isMessageInvalid"
        :aria-describedby="errors.message ? 'message-error' : undefined"
        rows="4"
        class="border rounded px-3 py-2 w-full"
      />
      <p
        v-if="errors.message"
        id="message-error"
        class="text-red-600 text-sm mt-1"
        role="alert"
      >
        {{ errors.message }}
      </p>
    </div>

    <button
      type="submit"
      class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700"
    >
      送信
    </button>
  </form>
</template>
```

### モーダルダイアログ

```vue
<script setup lang="ts">
import { ref, watch, onMounted, onUnmounted } from 'vue'

interface Props {
  isOpen: boolean
  title: string
}

interface Emits {
  (e: 'close'): void
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

const dialogRef = ref<HTMLElement | null>(null)

function handleClose() {
  emit('close')
}

function handleKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape') {
    handleClose()
  }
}

watch(() => props.isOpen, (isOpen) => {
  if (isOpen) {
    document.addEventListener('keydown', handleKeydown)
    // モーダルが開いたらフォーカスを移動
    setTimeout(() => {
      dialogRef.value?.focus()
    }, 100)
  } else {
    document.removeEventListener('keydown', handleKeydown)
  }
})

onUnmounted(() => {
  document.removeEventListener('keydown', handleKeydown)
})
</script>

<template>
  <Transition name="modal">
    <div
      v-if="isOpen"
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center"
      @click.self="handleClose"
      role="dialog"
      aria-modal="true"
      :aria-labelledby="title ? 'modal-title' : undefined"
    >
      <div
        ref="dialogRef"
        tabindex="-1"
        class="bg-white rounded-lg p-6 max-w-md w-full"
      >
        <div class="flex justify-between items-center mb-4">
          <h2 id="modal-title" class="text-xl font-bold">
            {{ title }}
          </h2>
          <button
            @click="handleClose"
            aria-label="閉じる"
            class="text-gray-500 hover:text-gray-700"
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              class="h-6 w-6"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              aria-hidden="true"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M6 18L18 6M6 6l12 12"
              />
            </svg>
          </button>
        </div>
        <div>
          <slot />
        </div>
      </div>
    </div>
  </Transition>
</template>

<style scoped>
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}
</style>
```

## パフォーマンス最適化

### 算出プロパティのメモ化

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Product {
  id: string
  name: string
  price: number
  category: string
}

const products = ref<Product[]>([
  { id: '1', name: '商品A', price: 1000, category: '食品' },
  { id: '2', name: '商品B', price: 2000, category: '電化製品' },
  { id: '3', name: '商品C', price: 1500, category: '食品' },
])

const selectedCategory = ref('食品')

// ✅ 良い例: 重い計算をcomputed()でメモ化
const filteredProducts = computed(() => {
  console.log('フィルタリング実行') // 依存関係が変わったときのみ実行
  return products.value.filter(p => p.category === selectedCategory.value)
})

const totalPrice = computed(() => {
  return filteredProducts.value.reduce((sum, p) => sum + p.price, 0)
})
</script>

<template>
  <div>
    <select v-model="selectedCategory">
      <option value="食品">食品</option>
      <option value="電化製品">電化製品</option>
    </select>

    <ul>
      <li v-for="product in filteredProducts" :key="product.id">
        {{ product.name }} - {{ product.price }}円
      </li>
    </ul>

    <p>合計: {{ totalPrice }}円</p>
  </div>
</template>
```

### 遅延読み込み（Lazy Loading）

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

// ✅ 良い例: コンポーネントを遅延読み込み
const HeavyChart = defineAsyncComponent(() => import('./components/HeavyChart.vue'))

const AsyncModal = defineAsyncComponent({
  loader: () => import('./components/Modal.vue'),
  loadingComponent: () => import('./components/Loading.vue'),
  errorComponent: () => import('./components/Error.vue'),
  delay: 200,
  timeout: 3000,
})
</script>

<template>
  <div>
    <Suspense>
      <template #default>
        <HeavyChart />
      </template>
      <template #fallback>
        <div>チャート読み込み中...</div>
      </template>
    </Suspense>
  </div>
</template>
```

### v-onceとv-memo

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const items = ref([1, 2, 3, 4, 5])
</script>

<template>
  <div>
    <!-- ✅ 良い例: 一度だけレンダリング -->
    <p v-once>この値は変わりません: {{ count }}</p>

    <!-- ✅ 良い例: メモ化による最適化 -->
    <ul>
      <li
        v-for="item in items"
        :key="item"
        v-memo="[item]"
      >
        アイテム {{ item }}
      </li>
    </ul>

    <button @click="count++">カウント: {{ count }}</button>
  </div>
</template>
```

## 状態管理（Pinia）

```typescript
// stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  // state
  const user = ref<User | null>(null)
  const loading = ref(false)

  // getters
  const isAuthenticated = computed(() => user.value !== null)
  const displayName = computed(() => {
    return user.value ? `${user.value.firstName} ${user.value.lastName}` : 'ゲスト'
  })

  // actions
  async function login(email: string, password: string) {
    loading.value = true
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      })
      user.value = await response.json()
    } finally {
      loading.value = false
    }
  }

  function logout() {
    user.value = null
  }

  return {
    user,
    loading,
    isAuthenticated,
    displayName,
    login,
    logout,
  }
})

interface User {
  id: string
  firstName: string
  lastName: string
  email: string
}
```

```vue
<!-- 使用例 -->
<script setup lang="ts">
import { useUserStore } from '@/stores/user'

const userStore = useUserStore()
</script>

<template>
  <div>
    <p v-if="userStore.isAuthenticated">
      ようこそ、{{ userStore.displayName }}さん
    </p>
    <button v-else @click="userStore.login('user@example.com', 'password')">
      ログイン
    </button>
  </div>
</template>
```

## テスト

### Vitestを使用した単体テスト

```typescript
// components/__tests__/Counter.spec.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import Counter from '../Counter.vue'

describe('Counter', () => {
  it('初期値が0であること', () => {
    const wrapper = mount(Counter)
    expect(wrapper.text()).toContain('カウント: 0')
  })

  it('ボタンクリックでカウントが増えること', async () => {
    const wrapper = mount(Counter)
    const button = wrapper.find('button')

    await button.trigger('click')
    expect(wrapper.text()).toContain('カウント: 1')

    await button.trigger('click')
    expect(wrapper.text()).toContain('カウント: 2')
  })

  it('propsを正しく受け取ること', () => {
    const wrapper = mount(Counter, {
      props: {
        initialCount: 10,
      },
    })
    expect(wrapper.text()).toContain('カウント: 10')
  })
})
```

## まとめ

Vueのベストプラクティスは、Composition APIの活用、型安全性、アクセシビリティ、パフォーマンス最適化のバランスです。

### チェックリスト

- [ ] Composition API（`<script setup>`）を使用
- [ ] TypeScriptで型定義
- [ ] propsとemitsを型安全に定義
- [ ] Composablesでロジックを再利用
- [ ] computedで重い計算をメモ化
- [ ] v-forには一意なkeyを指定
- [ ] フォームにラベルとエラーメッセージを適切に設定
- [ ] ARIA属性を適切に使用
- [ ] 遅延読み込みでパフォーマンス向上
- [ ] Vitestで単体テストを記述

## 参考リンク

- [Vue 3公式ドキュメント](https://vuejs.org/)
- [Vue 3 Composition API](https://vuejs.org/guide/extras/composition-api-faq.html)
- [Pinia公式ドキュメント](https://pinia.vuejs.org/)
- [Vue Test Utils](https://test-utils.vuejs.org/)

## 次のステップ

次章では、Next.jsのベストプラクティスについて詳しく解説します。
