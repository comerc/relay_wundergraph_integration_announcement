# Почему стоит взглянуть на Relay и GraphQL снова

Если вы давно следите за моей работой, то знаете, что одним из моих любимых пристрастий являются сравнения GraphQL, REST, tRPC и других технологий, в которых не упоминаются Relay и Fragments. В этом посте я объясню, почему я считаю Relay переломным моментом, как мы сделали ее в 100 раз проще в использовании и внедрении, и почему вам стоит обратить на нее внимание.

## Что делает Relay таким особым?
Перестаньте на мгновение думать об API с точки зрения сервера. Вместо этого подумайте о потреблении API с точки зрения интерфейса и о том, как вы можете создавать масштабируемые и удобные для поддержки приложения. Вот где сияет Relay, вот где вы можете увидеть значительную разницу между Relay, REST, tRPC и другими технологиями.

Если вы раньше не использовали Relay, вы, возможно, никогда не осознавали, насколько мощным может быть GraphQL в сочетании с Relay. В следующем разделе будет объяснено, почему.

В то же время многие люди боятся Relay, потому что у него крутая кривая обучения. Существует мнение, что Relay сложно настроить и использовать, и это в какой-то мере правда. Не должно требоваться докторских степеней, чтобы его использовать.

Вот почему мы создали интеграцию Relay первого класса в WunderGraph, которая работает как с NextJS, так и с чистым React (например, используя Vite). Мы хотим сделать Relay более доступным и легким в использовании. Такие основные функции, как отрисовка на стороне сервера (SSR), генерация статических сайтов (SSG), сохраненные запросы (persisted Operations) и отрисовка по мере получения (также известная как Suspense), встроены и работают из коробки.

Прежде чем мы погрузимся в то, как мы упростили использование Relay, давайте сначала посмотрим, что делает Relay таким особым.

### Размещение требований к данным с использованием фрагментов
Типичный шаблон выборки данных в приложениях, таких как NextJS, заключается в выборке данных в корневом компоненте и передаче их дочерним компонентам. С помощью фреймворка, такого как tRPC, вы определяете процедуру, которая выбирает все данные, необходимые для одной страницы, и передает их детям. Таким образом, вы неявно определяете требования к данным для компонента.

Допустим, у вас есть страница, которая отображает список блогов, и каждый блог имеет список комментариев.

В корневом компоненте вы бы выбрали блоги с комментариями и передали данные в компонент блога, который, в свою очередь, передает комментарии в компонент комментария.

Давайте проиллюстрируем это с помощью некоторого кода:

```tsx
// in pages/index.tsx
export default function Home({ posts }: { posts: Post[] }) {
  return (
    <div>
      {posts.map((post) => (
        <BlogPost key={post.id} post={post} />
      ))}
    </div>
  );
}

// in components/BlogPost.tsx
export default function BlogPost({ post }: { post: Post }) {
  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
      <div>
        {post.comments.map((comment) => (
          <Comment key={comment.id} comment={comment} />
        ))}
      </div>
    </div>
  );
}

// in components/Comment.tsx
export default function Comment({ comment }: { comment: Comment }) {
  return (
    <div>
      <h2>{comment.title}</h2>
      <p>{comment.content}</p>
    </div>
  );
}
```

В качестве примера, компонент `Comment` имеет две зависимости данных: `title` и `content`. Допустим, мы используем этот компонент в 10 разных местах нашего приложения. Если мы хотим добавить новое поле в компонент `Comment`, например, `author`, нам придется выяснить все места, где мы используем компонент `Comment`, перейти к корневому компоненту, найти процедуру, которая извлекает данные, и добавить в нее новое поле.

Вы можете видеть, как это быстро может стать огромной нагрузкой на поддержку. Проблема, которая приводит к этому, заключается в том, что мы извлекаем данные сверху вниз. Результатом является тесная связь между логикой извлечения данных и компонентами.

С Relay и Fragments мы можем разместить требования к данным с компонентом, одновременно отвязывая логику извлечения данных от компонента. Вместе с маскировкой данных (следующий раздел), это является прорывом, потому что это позволяет нам создавать повторно используемые компоненты, которые отделены от логики извлечения данных.

Стоит отметить, что сам GraphQL не решает эту проблему. Более того, большинство клиентов GraphQL не поощряют этот шаблон, что приводит к тем же проблемам, которые мы видели с REST API.

Так называемые "Бог-запросы", которые извлекают все данные для страницы, являются общим шаблоном для клиентов GraphQL. Без Fragments это действительно та же проблема, что и с REST API или tRPC, только с другим синтаксисом и добавленными накладными расходами GraphQL.

Давайте посмотрим, как мы можем достичь этого с помощью Relay и Fragments.

```tsx
// in pages/index.tsx
export default function Home() {
  const data = useFragment(
    graphql`
      query Home_posts on Query {
        posts {
          ...BlogPost_post
        }
      }
    `,
    null
  );

  return (
    <div>
      {data.posts.map((post) => (
        <BlogPost key={post.id} post={post} />
      ))}
    </div>
  );
}

// in components/BlogPost.tsx
export default function BlogPost({ post }: { post: Post }) {
  const data = useFragment(
    graphql`
      fragment BlogPost_post on Post {
        title
        content
        comments {
          ...Comment_comment
        }
      }
    `,
    post
  );

  return (
    <div>
      <h1>{data.title}</h1>
      <p>{data.content}</p>
      <div>
        {data.comments.map((comment) => (
          <Comment key={comment.id} comment={comment} />
        ))}
      </div>
    </div>
  );
}

// in components/Comment.tsx
export default function Comment({ comment }: { comment: Comment }) {
  const data = useFragment(
    graphql`
      fragment Comment_comment on Comment {
        title
        content
      }
    `,
    comment
  );

  return (
    <div>
      <h2>{data.title}</h2>
      <p>{data.content}</p>
    </div>
  );
}
```

В этом примере компонент `Comment` полностью отделен от логики извлечения данных. Он определяет свои требования к данным во фрагменте, который размещается вместе с компонентом. Мы можем использовать компонент `Comment` в любом количестве мест, он полностью отделен от логики извлечения данных.

Если мы хотим добавить новое поле в компонент `Comment`, например, поле `author`, мы можем просто добавить его в фрагмент, и компонент `Comment` автоматически получит новое поле.

Меняя нашу перспективу на логику извлечения данных, мы видим, что компонент `Home` не заботится о том, какие именно поля нужны компоненту `Comment`. Эта логика полностью отделена от компонента `Home` с помощью фрагментов.

Сказав это, есть еще одна вещь, которая делает возможными действительно отделенные компоненты: маскировка данных.

### Повторно используемые компоненты через маскировку данных
Допустим, у нас есть два соседних компонента, которые оба используют данные комментариев. Оба определяют свои требования к данным в отдельном фрагменте. Один компонент нуждается только в поле `title`, в то время как другой компонент требует поля `author` и `content`.

Если бы мы напрямую передали данные комментариев обоим компонентам, мы могли бы случайно использовать поле `title` в компоненте, которое не определило его в своем фрагменте. Таким образом, мы бы ввели зависимость между двумя компонентами.

Чтобы предотвратить это, Relay позволяет нам маскировать данные перед их передачей компоненту. Если компонент не определил поле в своем фрагменте, он не сможет получить к нему доступ, хотя оно теоретически доступно в данных.

На моем счету, ни один другой клиент API не имеет этой функции, поэтому я думаю, что вы не должны отказываться от GraphQL, не попробовав Relay. GraphQL и Relay стоят своих денег, если сравнивать их, например, с tRPC. Важно понимать преимущества, чтобы принять обоснованное решение о том, стоит ли это того.

Многие люди думают, что GraphQL и Relay полезны только для огромных приложений. Я думаю, что это заблуждение. Создание повторно используемых компонентов - это огромное преимущество для любого приложения, независимо от его размера. Если вы разобрались с фрагментами и маскировкой данных, вы действительно не захотите возвращаться к старому способу делать вещи.

Мы рассмотрим в следующем разделе, насколько легко мы сделали начало работы с Relay и фрагментами.

### Проверка и безопасность GraphQL во время компиляции
Еще одним преимуществом использования Relay является то, что "Relay Compiler" (недавно переписанный на Rust) компилирует, проверяет, оптимизирует и сохраняет все операции GraphQL на этапе сборки. С правильной настройкой мы можем полностью "удалить" API GraphQL из производственной среды. Это огромное преимущество для безопасности, потому что невозможно получить доступ к API GraphQL извне.

Кроме того, мы можем проверить все операции GraphQL на этапе сборки. Дорогостоящие операции, такие как нормализация и проверка, выполняются на этапе сборки, снижая накладные расходы во время выполнения.

## Как WunderGraph облегчает использование Relay?
Возможно, вы еще не убедились в преимуществах Relay, но я надеюсь, что вы хотя бы заинтересованы попробовать его.

Давайте посмотрим, как интеграция с WunderGraph облегчает начало работы с Relay.

### Настройка Relay + NextJS/Vite с WunderGraph проста
Мы сами пытались настроить Relay с NextJS и Vite. Это не просто. На самом деле, это довольно сложно. Мы нашли пакеты npm, которые пытаются преодолеть разрыв между Relay и NextJS, но они были не очень хорошо поддерживаемы, документация была устаревшей и, что самое главное, мы чувствовали, что они были слишком узкоспециализированными, например, принуждая использовать `getInitalProps`, который устарел в NextJS.

Поэтому мы отошли на шаг назад и создали решение, которое работает с Vanilla React и фреймворками для фронтенда, такими как NextJS и Vite, не будучи слишком узкоспециализированными. Мы создали необходимые инструменты, чтобы сделать отрисовку на стороне сервера (SSR), генерацию статических сайтов (SSG) и отрисовку по мере получения данных легкой в использовании с любым фреймворком для фронтенда.

Кроме того, мы убедились, что выбрали некоторые разумные значения по умолчанию, например, принудительное сохранение операций по умолчанию без какой-либо настройки, предоставляя пользователю безопасный по умолчанию опыт, не задумываясь об этом.

Итак, как выглядит простая настройка?

```tsx
// in pages/_app.tsx
import { WunderGraphRelayProvider } from '@/lib/wundergraph';
import '@/styles/globals.css';
import type { AppProps } from 'next/app';

export default function App({ Component, pageProps }: AppProps) {
	return (
		<WunderGraphRelayProvider initialRecords={pageProps.initialRecords}>
			<Component {...pageProps} />
		</WunderGraphRelayProvider>
	);
}
```

Вот и все. Все, что вам нужно сделать, это обернуть ваше приложение с помощью `WunderGraphRelayProvider` и передать prop `initialRecords`. Это работает с NextJS 12, 13, Vite и другими, так как не зависит от специфических для фреймворка API.

Далее, нам нужно настроить компилятор Relay для работы с WunderGraph. Как вы увидите, WunderGraph и Relay - это сочетание, созданное на небесах. Оба построены с учетом одних и тех же принципов: декларативность, типобезопасность, безопасность по умолчанию, локальность.

Relay является фронтенд-аналогом бэкенда WunderGraph. WunderGraph анализирует один или несколько GraphQL & REST API и представляет их в виде единой схемы GraphQL, которую мы называем виртуальным графом. Виртуальный, потому что мы на самом деле не представляем эту схему GraphQL внешнему миру. Вместо этого мы печатаем ее в файл, чтобы включить автозавершение в IDE и сделать ее доступной для компилятора Relay.

Во время выполнения мы не представляем API GraphQL внешнему миру. Вместо этого мы предоставляем только API RPC, который позволяет клиенту выполнять предварительно зарегистрированные операции GraphQL. Архитектура как WunderGraph, так и Relay делает интеграцию бесшовной.

> Кажется, что WunderGraph - это недостающий серверный аналог Relay.

### Конфигурация компилятора Relay с поддержкой сохраненных операций из коробки
Итак, как мы подключаем компилятор Relay для работы с WunderGraph?

Как уже упоминалось выше, WunderGraph автоматически сохраняет все операции GraphQL на этапе сборки. Для того чтобы это работало, нам нужно сообщить компилятору Relay, где "хранить" сохраненные операции. С другой стороны, Relay должен знать, где найти схему GraphQL. Поскольку WunderGraph хранит сгенерированную схему GraphQL в файле, все, что нам нужно сделать, это подключить их обоих, используя раздел `relay` в `package.json`.

```json
{
  "relay": {
    "src": "./src",
    "artifactDirectory": "./src/__relay__generated__",
    "language": "typescript",
    "schema": "./.wundergraph/generated/wundergraph.schema.graphql",
    "exclude": [
      "**/node_modules/**",
      "**/__mocks__/**",
      "**/__generated__/**",
      "**/.wundergraph/generated/**"
    ],
    "persistConfig": {
      "file": "./.wundergraph/operations/relay/persisted.json"
    },
    "eagerEsModules": true
  }
}
```

С этой конфигурацией компилятор Relay будет собирать все операции GraphQL в каталоге `./src`, генерировать типы TypeScript и сохранять сохраненные операции в `./.wundergraph/operations/relay/persisted.json`. Каждая сохраненная операция - это пара уникального ID (хеш) и операции GraphQL. WunderGraph автоматически прочитает этот файл, расширит его в `.graphql` файлы и сохранит их в `./.wundergraph/operations/relay/`, что автоматически зарегистрирует их как конечные точки JSON-RPC.

Кроме того, генератор кода WunderGraph сгенерирует для вас `WunderGraphRelayEnvironment`, который внутренне реализует fetch для выполнения вызовов RPC к API WunderGraph.

Вот сокращенная версия внутренностей:

```tsx
const fetchQuery: FetchFunction = async (params, variables) => {
		const { id, operationKind } = params;
		const response =
			operationKind === 'query'
				? await client.query({
						operationName: `relay/${id}`,
						input: variables,
				  })
				: await client.mutate({
						operationName: `relay/${id}`,
						input: variables,
				  });
		return {
			...response,
			errors: response.error ? [response.error] : [],
		};
	};
```

Функция `fetchQuery` создает запросы JSON-RPC из ID операции и переменных, на этом этапе GraphQL не участвует.

### Отрисовка на стороне сервера (SSR) с NextJS, Relay и WunderGraph
Теперь, когда мы настроили компилятор Relay, мы можем начать интеграцию Relay в наше приложение NextJS, например, с отрисовкой на стороне сервера (SSR).

```tsx
import { graphql } from 'react-relay';
import { pagesDragonsQuery as PagesDragonsQueryType } from '../__relay__generated__/pagesDragonsQuery.graphql';
import { Dragon } from '@/components/Dragon';
import { fetchWunderGraphSSRQuery } from '@/lib/wundergraph';
import { InferGetServerSidePropsType } from 'next';

const PagesDragonsQuery = graphql`
	query pagesDragonsQuery {
		spacex_dragons {
			...Dragons_display_details
		}
	}
`;

export async function getServerSideProps() {
	const relayData = await fetchWunderGraphSSRQuery<PagesDragonsQueryType>(PagesDragonsQuery);

	return {
		props: relayData,
	};
}

export default function Home({ queryResponse }: InferGetServerSidePropsType<typeof getServerSideProps>) {
    return (
        <div>
            {queryResponse.spacex_dragons.map((dragon) => (
                <Dragon key={dragon.id} dragon={dragon} />
            ))}
        </div>
    );
}
```

### Отрисовка по мере получения с Vite, Relay и WunderGraph
Вот еще один пример использования Vite с отрисовкой по мере получения данных:

```tsx
import { graphql, loadQuery } from 'react-relay';
import { getEnvironment, useLiveQuery } from '../lib/wundergraph';
import { Dragon } from './Dragon';
import { DragonsListDragonsQuery as DragonsListDragonsQueryType } from '../__relay__generated__/DragonsListDragonsQuery.graphql';

const AppDragonsQuery = graphql`
	query DragonsListDragonsQuery {
		spacex_dragons {
			...Dragons_display_details
		}
	}
`;

const dragonsListQueryReference = loadQuery<DragonsListDragonsQueryType>(getEnvironment(), AppDragonsQuery, {});

export const DragonsList = () => {
	const { data } = useLiveQuery<DragonsListDragonsQueryType>({
		query: AppDragonsQuery,
		queryReference: dragonsListQueryReference,
	});

	return (
		<div>
			<p>Dragons:</p>
			{data?.spacex_dragons?.map((dragon, dragonIndex) => {
				if (dragon) return <Dragon key={dragonIndex.toString()} dragon={dragon} />;
				return null;
			})}
		</div>
	);
};
```

## Заключение
Вашим ключевым выводом должно быть то, что GraphQL и Relay приносят много ценности. Вместе с WunderGraph вы можете создавать современные полнофункциональные приложения на основе трех прочных столпов:

- Размещение компонентов и требований к данным
- Отделенные повторно используемые компоненты с использованием маскировки данных
- Валидация и безопасность на этапе компиляции

Более того, с этим стеком вы действительно не ограничены только GraphQL API и React. Возможно использование Relay с REST API, или даже SOAP, и мы также не ограничены React, так как Relay - это всего лишь библиотека для извлечения данных.

Если вы хотите узнать больше о WunderGraph, ознакомьтесь с [документацией](https://docs.wundergraph.com/).

Хотите попробовать некоторые примеры?

- [NextJS Relay](https://github.com/wundergraph/wundergraph/tree/main/examples/nextjs-relay) 
- [NextJS Relay Todo App](https://github.com/wundergraph/wundergraph/tree/main/examples/nextjs-relay-todos)
- [Vite Relay](https://github.com/wundergraph/wundergraph/tree/main/examples/vite-react-relay)

Еще одна вещь. Это действительно только начало нашего пути к тому, чтобы сделать мощь GraphQL и Relay доступной для всех. Оставайтесь на связи в [Twitter](https://twitter.com/TheWorstFounder) или присоединяйтесь к нашему сообществу [Discord](https://wundergraph.com/discord), чтобы быть в курсе, так как мы скоро запустим что-то действительно захватывающее, что поднимет это на новый уровень.