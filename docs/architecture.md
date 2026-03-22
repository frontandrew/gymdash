# Архитектура фронтенд-приложения

## Содержание

1. [Цели и принципы](#1-цели-и-принципы)
2. [Технологический стек](#2-технологический-стек)
3. [Методология: Feature-Sliced Design](#3-методология-feature-sliced-design)
4. [Структура проекта](#4-структура-проекта)
5. [Слои и их ответственность](#5-слои-и-их-ответственность)
6. [Правила зависимостей](#6-правила-зависимостей)
7. [Разделение логики и представления](#7-разделение-логики-и-представления)
8. [Инструменты и конфигурация](#8-инструменты-и-конфигурация)
9. [Соглашения об именовании](#9-соглашения-об-именовании)

---

## 1. Цели и принципы

Архитектура проекта строится вокруг трёх основных целей:

**Разделение логики и представления.** Бизнес-логика, работа с состоянием и обращения к API не должны смешиваться с компонентами Vue. Компоненты отвечают исключительно за рендеринг и передачу событий.

**Отсутствие циклических зависимостей.** Зависимости между модулями направлены строго в одну сторону — от высокоуровневых слоёв к низкоуровневым. Ни один модуль не может импортировать из слоя, который стоит выше него в иерархии.

**Формализованные архитектурные границы.** Структура проекта является контрактом. Правила зависимостей закреплены в конфигурации ESLint и проверяются автоматически при каждом коммите и в CI.

### Принципы

- **Явное лучше неявного.** Каждый модуль экспортирует только то, что предназначено для внешнего использования, через публичный `index.ts`.
- **Один модуль — одна ответственность.** Фича решает одну пользовательскую задачу. Сущность описывает один доменный объект.
- **Зависимости идут вниз.** Верхние слои знают о нижних, но не наоборот.
- **Переиспользование через нижние слои.** Если логика нужна в двух фичах, она опускается в `entities` или `shared`.

---

## 2. Технологический стек

| Категория | Инструмент | Назначение |
|---|---|---|
| UI-фреймворк | Vue 3 (Composition API) | Компонентный рендеринг |
| Язык | TypeScript | Статическая типизация |
| Сборщик | Vite | Dev-сервер и production-сборка |
| Управление состоянием | Pinia | Глобальные stores |
| Серверное состояние | TanStack Query | Кеширование запросов к API |
| Платформа | PWA | Мобильный браузер, офлайн, Push API |

---

## 3. Методология: Feature-Sliced Design

Проект следует методологии [Feature-Sliced Design (FSD)](https://feature-sliced.design/). FSD определяет иерархию слоёв и правила импортов между ними, что решает задачу управления зависимостями на уровне файловой структуры.

### Ключевые понятия

**Слой (layer)** — уровень иерархии (`app`, `pages`, `widgets`, `features`, `entities`, `shared`). Каждый слой может импортировать только из слоёв ниже себя.

**Слайс (slice)** — изолированная единица внутри слоя (например, `features/auth`, `entities/user`). Слайсы одного слоя не импортируют друг друга.

**Сегмент (segment)** — тип артефакта внутри слайса: `ui`, `model`, `api`, `lib`, `config`.

---

## 4. Структура проекта

```
src/
├── app/                        # Инициализация приложения
│   ├── providers/              # Глобальные провайдеры (router, query client)
│   ├── styles/                 # Глобальные CSS / design tokens
│   └── index.ts
│
├── pages/                      # Страницы — точки входа маршрутов
│   ├── home/
│   │   ├── ui/
│   │   │   └── HomePage.vue
│   │   └── index.ts
│   └── ...
│
├── widgets/                    # Самодостаточные блоки UI из нескольких фич
│   ├── header/
│   │   ├── ui/
│   │   │   └── AppHeader.vue
│   │   └── index.ts
│   └── ...
│
├── features/                   # Пользовательские сценарии
│   ├── auth/
│   │   ├── ui/                 # Компоненты фичи
│   │   │   └── LoginForm.vue
│   │   ├── model/              # Store, composables, бизнес-логика
│   │   │   ├── auth.store.ts
│   │   │   └── useLogin.ts
│   │   ├── api/                # Запросы к API
│   │   │   └── auth.api.ts
│   │   └── index.ts            # Публичный API слайса
│   └── ...
│
├── entities/                   # Доменные объекты
│   ├── user/
│   │   ├── ui/                 # Отображение сущности
│   │   │   └── UserCard.vue
│   │   ├── model/              # Типы, store, трансформации
│   │   │   ├── user.types.ts
│   │   │   └── user.store.ts
│   │   └── index.ts
│   └── ...
│
└── shared/                     # Переиспользуемый код без доменной привязки
    ├── api/                    # Базовый HTTP-клиент, конфиг TanStack Query
    ├── ui/                     # Базовые UI-компоненты (Button, Input, Modal)
    ├── lib/                    # Утилиты (formatDate, validators)
    ├── config/                 # Константы, переменные окружения
    └── types/                  # Глобальные типы и интерфейсы
```

---

## 5. Слои и их ответственность

### `shared`

Базовый слой. Содержит всё, что не привязано к домену и может использоваться в любом слое.

- **`shared/api`** — базовый HTTP-клиент (например, `axios`-инстанс), конфигурация TanStack Query (`QueryClient`), типы ответов API.
- **`shared/ui`** — атомарные компоненты (кнопки, поля ввода, иконки). Не содержат бизнес-логики.
- **`shared/lib`** — чистые функции-утилиты.
- **`shared/config`** — константы окружения (`import.meta.env`), роуты.

### `entities`

Доменные объекты: модели, их отображение и базовые операции над ними.

- **`model/`** — TypeScript-типы сущности, Pinia store с серверным состоянием через TanStack Query.
- **`ui/`** — компоненты для отображения сущности (`UserAvatar`, `OrderStatus`). Не содержат логики взаимодействия.
- **`api/`** — функции запросов, специфичных для сущности.

### `features`

Пользовательские сценарии — всё, что пользователь _делает_ в приложении.

- **`model/`** — composables с бизнес-логикой, мутации TanStack Query, вспомогательные stores.
- **`ui/`** — компоненты с формами и интерактивными элементами. Вся логика делегируется в `model/`.
- **`api/`** — API-запросы, специфичные для данного сценария.

### `widgets`

Самодостаточные блоки интерфейса, объединяющие несколько фич и сущностей в единый UI-блок. Не содержат собственной бизнес-логики.

### `pages`

Компонуют виджеты и фичи в страницы, привязанные к маршрутам. Содержат минимум логики — только компоновку и передачу роут-параметров.

### `app`

Точка входа. Инициализирует приложение: подключает Vue Router, Pinia, QueryClient, PWA-плагин, глобальные стили.

---

## 6. Правила зависимостей

### Направление зависимостей

```
app → pages → widgets → features → entities → shared
```

Каждый слой может импортировать только из слоёв ниже. Обратные импорты запрещены.

### Запреты между слайсами

Слайсы одного слоя изолированы друг от друга. Нельзя:

```ts
// features/auth/model/useLogin.ts
import { usePayment } from '@/features/payment' // ❌ feature → feature запрещено
```

Если двум фичам нужна общая логика, она выносится в `entities` или `shared`.

### Публичный API через index.ts

Каждый слайс предоставляет только явно экспортированные артефакты через `index.ts`. Прямые импорты внутренних файлов запрещены:

```ts
import { useLogin } from '@/features/auth'           // ✅ через публичный API
import { useLogin } from '@/features/auth/model/useLogin' // ❌ прямой импорт внутренностей
```

---

## 7. Разделение логики и представления

Компоненты Vue (`*.vue`) не содержат бизнес-логики, прямых вызовов API или сложных вычислений. Вся логика инкапсулируется в composables и stores.

### Пример: фича авторизации

**`features/auth/model/useLogin.ts`** — логика:

```ts
import { useMutation } from '@tanstack/vue-query'
import { authApi } from '../api/auth.api'
import { useUserStore } from '@/entities/user'

export function useLogin() {
  const userStore = useUserStore()

  const { mutate, isPending, isError } = useMutation({
    mutationFn: authApi.login,
    onSuccess(data) {
      userStore.setUser(data.user)
    },
  })

  return { login: mutate, isPending, isError }
}
```

**`features/auth/ui/LoginForm.vue`** — представление:

```vue
<script setup lang="ts">
import { useLogin } from '../model/useLogin'

const { login, isPending, isError } = useLogin()
</script>

<template>
  <form @submit.prevent="login(credentials)">
    <!-- только рендеринг и события -->
  </form>
</template>
```

Компонент знает только о том, что нужно отобразить и какие события передать. Всё остальное — в `model/`.

---

## 8. Инструменты и конфигурация

### ESLint

Архитектурные правила автоматически проверяются через `@feature-sliced/eslint-plugin`:

```js
// eslint.config.js
import { defineConfig } from 'eslint/config'
import fsd from '@feature-sliced/eslint-plugin'

export default defineConfig([
  fsd.configs.recommended,
  {
    rules: {
      // Запрет прямых импортов внутренних файлов слайса
      '@feature-sliced/public-api': 'error',
      // Запрет импортов в обход иерархии слоёв
      '@feature-sliced/layers-slices': 'error',
    },
  },
])
```

Дополнительно подключается `import/no-cycle` для обнаружения циклических зависимостей:

```js
import importPlugin from 'eslint-plugin-import'

export default defineConfig([
  {
    plugins: { import: importPlugin },
    rules: {
      'import/no-cycle': 'error',
    },
  },
])
```

### TypeScript Path Aliases

Для читаемых импортов без относительных путей:

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

```ts
// vite.config.ts
import { resolve } from 'path'

export default defineConfig({
  resolve: {
    alias: { '@': resolve(__dirname, 'src') },
  },
})
```

### PWA

Настройка через `vite-plugin-pwa`. Service Worker обрабатывает офлайн-кеш, Push-уведомления регистрируются через `Push API` в слое `shared/lib/push.ts`.

---

## 9. Соглашения об именовании

| Артефакт | Конвенция | Пример |
|---|---|---|
| Компоненты Vue | PascalCase | `LoginForm.vue`, `UserCard.vue` |
| Composables | camelCase с префиксом `use` | `useLogin.ts`, `useGeolocation.ts` |
| Pinia stores | camelCase с суффиксом `store` | `user.store.ts`, `auth.store.ts` |
| API-функции | camelCase с суффиксом `api` | `auth.api.ts`, `order.api.ts` |
| Типы / интерфейсы | PascalCase | `User`, `AuthResponse`, `OrderStatus` |
| Файлы утилит | kebab-case | `format-date.ts`, `validators.ts` |
| Папки слайсов | kebab-case | `features/user-profile/` |
| Публичный API слайса | всегда `index.ts` | `features/auth/index.ts` |