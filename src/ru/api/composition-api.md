# Composition API

> В этом разделе в примерах кода используется синтаксис [однофайловых компонентов](../guide/single-file-component.md)

## `setup`

Опция компонента, которая является стартовой точкой для Composition API и выполняется **перед созданием** компонента, после разрешения входных параметров `props`.

- **Аргументы:**

  - `{Data} props` — входные параметры
  - `{SetupContext} context` — контекст

- **Типизация:**

  ```ts
  interface Data {
    [key: string]: unknown
  }

  interface SetupContext {
    attrs: Data
    slots: Slots
    emit: (event: string, ...args: unknown[]) => void
  }

  function setup(props: Data, context: SetupContext): Data
  ```

  :::tip Совет
  Необходимо использовать [defineComponent](global-api.md#definecomponent) чтобы получить вывод типов для аргументов, передаваемых в `setup()`.
  :::

- **Пример**

  С использованием шаблона:

  ```vue
  <!-- MyBook.vue -->
  <template>
    <div>{{ readersNumber }} {{ book.title }}</div>
  </template>

  <script>
    import { ref, reactive } from 'vue'

    export default {
      setup() {
        const readersNumber = ref(0)
        const book = reactive({ title: 'Руководство по Vue 3' })

        // объявление для шаблона
        return {
          readersNumber,
          book
        }
      }
    }
  </script>
  ```

  С использованием render-функции:

  ```js
  // MyBook.vue

  import { h, ref, reactive } from 'vue'

  export default {
    setup() {
      const readersNumber = ref(0)
      const book = reactive({ title: 'Руководство по Vue 3' })

      // Обратите внимание, здесь потребуется явно объявлять значение ref
      return () => h('div', [readersNumber.value, book.title])
    }
  }
  ```

- **См. также**: [Composition API `setup`](../guide/composition-api-setup.md)

## Хуки жизненного цикла

Хуки жизненного цикла можно регистрировать через явно импортируемые функции `onX`:

```js{1,5,8,11}
import { onMounted, onUpdated, onUnmounted } from 'vue'

const MyComponent = {
  setup() {
    onMounted(() => {
      console.log('примонтирован!')
    })
    onUpdated(() => {
      console.log('обновлён!')
    })
    onUnmounted(() => {
      console.log('размонтирован!')
    })
  }
}
```

Функции хуков жизненного цикла могут использоваться только синхронно внутри [`setup()`](#setup), поскольку они полагаются на внутреннее локальное состояние для определения текущего активного экземпляра (экземпляра компонента чей `setup()` сейчас вызывается). Их вызов без текущего активного экземпляра приведёт к ошибке.

Контекст экземпляра компонента также устанавливается во время синхронного выполнения хуков жизненного цикла. В результате, методы-наблюдатели и вычисляемые свойства созданные синхронно внутри хуков жизненного цикла будут также автоматически уничтожаться при размонтировании компонента.

- **Соответствие между хуками Options API и Composition API**

  - ~~`beforeCreate`~~ -> используйте `setup()`
  - ~~`created`~~ -> используйте `setup()`
  - `beforeMount` -> `onBeforeMount`
  - `mounted` -> `onMounted`
  - `beforeUpdate` -> `onBeforeUpdate`
  - `updated` -> `onUpdated`
  - `beforeUnmount` -> `onBeforeUnmount`
  - `unmounted` -> `onUnmounted`
  - `errorCaptured` -> `onErrorCaptured`
  - `renderTracked` -> `onRenderTracked`
  - `renderTriggered` -> `onRenderTriggered`
  - `activated` -> `onActivated`
  - `deactivated` -> `onDeactivated`

- **См. также**: [Composition API. Хуки жизненного цикла](../guide/composition-api-lifecycle-hooks.md)

## Provide / Inject

Функции `provide` и `inject` предоставляют возможность внедрения зависимостей. Они могут быть вызваны только во время [`setup()`](#setup) с текущим активным экземпляром.

- **Типизация:**

  ```ts
  interface InjectionKey<T> extends Symbol {}

  function provide<T>(key: InjectionKey<T> | string, value: T): void

  // без значения по умолчанию
  function inject<T>(key: InjectionKey<T> | string): T | undefined

  // со значением по умолчанию
  function inject<T>(key: InjectionKey<T> | string, defaultValue: T): T

  // с функцией-фабрикой
  function inject<T>(
    key: InjectionKey<T> | string,
    defaultValue: () => T,
    treatDefaultAsFactory: true
  ): T
  ```

  Vue предоставляет интерфейс `InjectionKey` — общий тип, расширяющий `Symbol`. Его можно использовать для синхронизации типа внедряемого значения между провайдером и получателем:

  ```ts
  import { InjectionKey, provide, inject } from 'vue'

  const key: InjectionKey<string> = Symbol()

  provide(key, 'foo') // указание не-строчного значения приведёт к ОШИБКЕ

  const foo = inject(key) // тип foo: string | undefined
  ```

  При использовании строковых ключей или не-типизированных Symbol, тип внедряемого значения необходимо объявить явно:

  ```ts
  const foo = inject<string>('foo') // string | undefined
  ```

- **См. также**:
  - [Provide / Inject](../guide/component-provide-inject.md)
  - [Composition API. Provide / Inject](../guide/composition-api-provide-inject.md)

## `getCurrentInstance`

Метод `getCurrentInstance` позволяет получить доступ к внутреннему экземпляру компонента, что пригодится для продвинутых пользователей или разработчиков библиотек.

```ts{1,5,7}
import { getCurrentInstance } from 'vue'

const MyComponent = {
  setup() {
    const internalInstance = getCurrentInstance()

    internalInstance.appContext.config.globalProperties // доступ к globalProperties
  }
}
```

Метод `getCurrentInstance` **работает только** во время [setup](#setup) или [хуков жизненного цикла](#хуки-жизненного-цикла)

> При использовании вне [setup](#setup) или [хуков жизненного цикла](#хуки-жизненного-цикла) вызывайте `getCurrentInstance()` в `setup` и используйте вместо него экземпляр.

```ts
const MyComponent = {
  setup() {
    const internalInstance = getCurrentInstance() // работает

    const id = useComponentId() // работает

    const handleClick = () => {
      getCurrentInstance() // НЕ БУДЕТ РАБОТАТЬ
      useComponentId() // НЕ БУДЕТ РАБОТАТЬ

      internalInstance // работает
    }

    onMounted(() => {
      getCurrentInstance() // работает
    })

    return () =>
      h(
        'button',
        {
          onClick: handleClick
        },
        `uid: ${id}`
      )
  }
}

// также работает при вызове в функции композиции
function useComponentId() {
  return getCurrentInstance().uid
}
```
