---
title: Сервер
description: Серверные возможности Solid
sort: 3
---

# Рендеринг на стороне сервера (`SSR`)

Solid обрабатывает серверный рендеринг, компилируя шаблоны JSX в сверхэффективный код добавления строк. Это может быть достигнуто с помощью плагина babel, с помощью аргумента `generate: 'ssr'`. Для того, чтобы код мог подключиться после отправки с сервера на клиенте вам необходимо передать `hydratable: true`, чтобы сгенерировать код, с [гидрацией](https://github.com/gatsbyjs/gatsby-ru/blob/master/docs/docs/react-hydration.md).

Среды выполнения solid-js и solid-js/web меняются местами на нереактивные аналоги при работе с Node. Для других сред вам нужно будет связать код сервера с условным экспортом, установленным как `node`. У большинства сборщиков кода (`bundlers`) есть способ сделать это. В общем, мы также рекомендуем использовать условия экспорта `solid`, а также рекомендуется, чтобы библиотеки поставляли свой исходный код с экспортом `solid`.

Сборка для SSR определенно требует немного большей настройки, потому что мы будем генерировать 2 отдельных бандла. В записи клиента следует использовать `hydrate`:

```jsx
import { hydrate } from 'solid-js/web'

hydrate(() => <App />, document)
```

_Примечание: рендеринг и гидрация должны происходить в главном компоненте (`Entry`). Это позволяет нам полностью описать наш интерфейс в JSX._

Сервер может использовать один из четырех вариантов рендеринга. Каждый вариант также создает JS код, который вставляется в заголовок документа.

```jsx
import { renderToString, renderToStringAsync, renderToNodeStream, renderToWebStream } from 'solid-js/web'

// Синхронный рендеринг строки
const html = renderToString(() => <App />)

// Асинхронный рендеринг строки
const html = await renderToStringAsync(() => <App />)

// API-интерфейс Node Stream
pipeToNodeWritable(App, res)

// API веб-потока (например, Cloudflare Workers)
const { readable, writable } = new TransformStream()
pipeToWritable(() => <App />, writable)
```

Для вашего удобства solid-js/web экспортирует флаг `isServer`. Это полезно еще и для бандлеров, если нужно что-то выполнить на стороне только клиента/сервера.

```jsx
import { isServer } from 'solid-js/web'

if (isServer) {
  // Только на сервере
} else {
  // Только в браузере
}
```

## Скрипт гидрации

Для постепенной гидрации даже до загрузки среды выполнения Solid на страницу необходимо вставить специальный скрипт. Его можно либо сгенерировать и вставить через `generateHydrationScript`, либо включить как часть JSX с помощью тега`<HydrationScript />`.

```js
import { generateHydrationScript } from 'solid-js/web'

const app = renderToString(() => <App />)

const html = `
  <html lang="en">
    <head>
      <title>🔥 Solid SSR 🔥</title>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />
      <link rel="stylesheet" href="/styles.css" />
      ${generateHydrationScript()}
    </head>
    <body>${app}</body>
  </html>
`
```

```jsx
import { HydrationScript } from 'solid-js/web'

const App = () => {
  return (
    <html lang='en'>
      <head>
        <title>🔥 Solid SSR 🔥</title>
        <meta charset='UTF-8' />
        <meta name='viewport' content='width=device-width, initial-scale=1.0' />
        <link rel='stylesheet' href='/styles.css' />
        <HydrationScript />
      </head>
      <body>{/*... Приложение */}</body>
    </html>
  )
}
```

При гидрации из документа некоторые ресурсы, которые будут недоступны в клиентской части, также могут выдать ошибку. Solid предоставляет компонент `<NoHydration>`, дочерние элементы которого будут нормально работать на сервере, но не будут гидрироваться в браузере.

```jsx
<NoHydration>
  {manifest.map(m => (
    <link rel='modulepreload' href={m.href} />
  ))}
</NoHydration>
```

## Асинхронный и потоковый SSR

Эти механизмы основаны на знании Solid о том, как работает ваше приложение. Это достигается за счет использования `Suspense` и API `Ресурсов` на сервере вместо предварительной выборки и последующего рендеринга. Solid загружает компоненты по мере их доступности, как на сервере, так и на клиенте. Ваш код и шаблоны выполнения написаны точно так же.

Асинхронный рендеринг ждет, пока все границы `Suspense` не выполнятся, а затем отправляет результаты (или записывает их в файл в случае создания статического сайта).

Потоковая передача начинает сбрасывать синхронный контент в браузер, немедленно запуская рендеринг `fallback` копонентов вашего `Suspense` на сервере. Затем, когда асинхронные данные заканчиваются на сервере, он отправляет данные в том же потоке клиенту, чтобы успешно завершить процесс `Suspense`, где браузер заменит все `fallback` на реальный контент.

Преимущество такого подхода:

- Серверу не нужно ждать ответа от асинхронных данных. `Ресурсы` начнут загружаться в браузере раньше, и пользователь, соответственно, раньше увидит контент.
- По сравнению с клиентской выборкой, такой как `JAMStack`, загрузка данных начинается на сервере немедленно и не требует ожидания загрузки клиентского JavaScript.
- Все данные сериализуются и автоматически передаются от сервера к клиенту.

## Предостережения по SSR

Решение Solid's для `Isomorphic SSR` очень мощное, поскольку вы можете писать свой код универсально, и он будет работать одинаково в обеих средах. Однако есть ожидания, что это приведет к гидрации. В основном визуализированный вид на клиенте такой же, как и на сервере. Необязательно, чтобы текст был точным, но структурно разметка должна быть такой же.

Мы используем маркеры, отображаемые на сервере, для сопоставления элементов и местоположений ресурсов на сервере. По этой причине Клиент и Сервер должны иметь одинаковые компоненты. Обычно это не проблема, учитывая, что Solid визуализирует одинаково на клиенте и сервере. Но в настоящее время нет средств для рендеринга на сервере чего-то, что не гидрировалось бы на клиенте. В настоящее время нет возможности частично гидрировать всю страницу и не создавать для нее маркеры гидрации. Все или ничего. Частичная гидрация (`Partial Hydration`) - это то, что мы хотим изучить в будущем.

Наконец, все ресурсы должны быть определены в дереве рендеринга. Они автоматически сериализуются и обрабатываются в браузере, однако это работает, потому что методы `render` или `pipeTo` отслеживают ход визуализации. Это то, чего мы не можем сделать, если они созданы в изолированном контексте. Точно так же на сервере нет реактивности, поэтому не обновляйте `Сигналы` при первоначальном рендеринге и ожидайте, что они будут отражаться выше по дереву. Хотя у нас есть границы приостановки, SSR Solid в основном работает сверху вниз.

## Начало работы с SSR

Конфигурации SSR непростые. У нас есть несколько примеров в пакете [solid-ssr](https://github.com/solidjs/solid/blob/main/packages/solid-ssr).

Однако в разработке находится новое средство для старта проекта [SolidStart](https://github.com/solidjs/solid-start), которое призвано сделать этот опыт более плавным.

## Начало работы с созданием статических сайтов

[solid-ssr](https://github.com/solidjs/solid/blob/main/packages/solid-ssr) также поставляется с простой утилитой для создания статических или предварительно отрисованных сайтов. Прочтите README для получения дополнительной информации.