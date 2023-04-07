---
category: 2023
tags: ["2023", "dev", "typescript", "generator", "javascript", "async"]
---

# Timeout이 있는 자바스크립트 제네레이터

제네레이터에 대해서는 [자바스크립트 제네레이터](/2023/JavaScript-Generator)를 참고하는 게 도움이 될 것이다.

GeoJSON 등 지리정보(GIS)를 처리하는 UI를 설계하다 보면 해당 지역에 있는 모든 POI를 출력할 필요가 없을 때가 많다. 특히 사용자가 지도를 패닝 Panning 하고 있을 때는 POI의 일부만 보여주어도 충분한 경우가 많다. 그렇다고 아무 것도 보여주지 않기에는 뭔가 허전하기에 패닝이나 스크롤 이벤트가 일어나기 전까지는 로드할 수 있는 최대한의 POI를 로드하는 함수를 작성해 보았다.

핵심 코드만 남기기 위해 이벤트 핸들러 등 DOM과 관련된 코드는 모두 제거하고 인터럽트는 간단히 타이머 함수를 사용하였다.

## JavaScript

[자바스크립트 데모 보기](https://codesandbox.io/s/timeout-generator-y6okhw?file=/src/index.js){: .btn .btn--info}{:target="\_blank"}

```javascript
const delay = (msec) => new Promise((done) => setTimeout(() => done(), msec));
```

이 함수는 따로 설명이 필요 없을 것이다. 지정된 밀리초동안 실행을 지연시키는 함수이다.

```javascript
async function* slowEmitter(arr, waitTime) {
  for (let i = 0; i < arr.length; i++) {
    yield arr[i];
    await delay(waitTime);
  }
}

async function* slowFetcher(arr, fetchIntervalTime) {
  for (let i = 0; i < arr.length; i++) {
    const fetcher = fetch(
      `https://jsonplaceholder.typicode.com/todos/${arr[i]}`
    ).then((res) => res.json());

    const result = await Promise.all([fetcher, delay(fetchIntervalTime)]);

    yield result[0];
  }
}
```

`slowFetcher는` 좀 복잡하기 때문에 `slowEmitter`를 먼저 보는 게 좋다.
arr은 일종의 API호출 파라미터라고 보면 된다. 정확히는 파라미터의 리스트이다. POI를 요청하는 endpoint는 하나이고 내부의 파라미터만 변경되기 때문에 이런 식으로 만들었다. 만약 POI Get API가 배열을 반환한다면 가급적 최소한의 정보만을 받아오도록(=최소한의 통신 부하를 주도록) 하고 slowFetcher에서는 받아 온 리스트를 가지고 getDetail 같은 것을 돌려주도록 설계하면 된다. 리스트를 가져올 때 우선순위로 정렬할 수 있다면 더 좋다.

`slowFetcher`의 `Promise.all` 함수는 API호출 속도를 고의로 지연시키기 위해 넣었다. API가 아무리 빨라도 `fetchIntervalTime`에 지정된 시간(밀리초)만큼은 기다려야 한다. 만약 API가 이것보다 느리면 그만큼의 지연이 걸린다. slowFetcher는 타임아웃과 관련된 함수가 아니므로 API에서 지연이 걸리면 걸리는 만큼 대기하도록 설계하였다.

```javascript
async function waitUntil(generator, timeWait) {
  const arr = [];
  const workerFn = async () => {
    for await (const obj of generator) {
      arr.push(obj);
    }
  };

  await Promise.race([workerFn(), delay(timeWait)]);
  return arr;
}
```

이것이 타임아웃과 관련된 함수이다. 제네레이터를 `for await`를 사용해 순회하면서 타임아웃이 걸릴 때까지 최대한 API를 호출한다. 그리고 타임아웃이 걸리면, 그때까지 처리된 리스트를 돌려준다.

여기서는 `arr` 배열에 넣었다가 한번에 돌려줬지만 실제 UI로직에 이를 적용할 때에는 UI에 해당 POI를 배치하는 코드가 들어갸게 된다. 함수의 원래 목적이 타임아웃 전까지 **"보여줄 수 있는 건 최대한 보여준다"** 이므로 하나 처리될 때마다 UI를 갱신해 주어야 하는 것이다. 더 최적화하고 싶다면 `setInterval`이나 `requestAnimationFrame`함수를 중간에 끼워줄 수도 있을테지만, 아마 그 정도 최적화는 프레임워크가 이미 해 놨을 것이다.

이제 이 자바스크립트 코드의 타입스크립트 버전을 볼 차례다.

## TypeScript

[타입스크립트 데모 보기](https://codesandbox.io/s/timeout-generator-ts-zvzivq){: .btn .btn--info}{:target="\_blank"}

```typescript
type GeneratorFn<T> = AsyncGenerator<T, void, void>;
```

`slowEmitter`, `slowFetcher` 모두 `AsyncGenerator` 타입을 확장했다. `AsyncGenerator`의 [타입 시그니처](https://microsoft.github.io/PowerBI-JavaScript/interfaces/_node_modules_typedoc_node_modules_typescript_lib_lib_es2018_asyncgenerator_d_.asyncgenerator.html)는 `Interface AsyncGenerator<T, TReturn, TNext>` 이다.

```
T: yield가 돌려줄 타입
TReturn: 제네레이터가 끝났을 때 돌려줄 타입(return type)
TNext: yield의 반환값 타입
```

우리가 작성한 제네레이터는 yield로 데이터를 보내만 줄 뿐, 완료 시의 동작도 없고 제네레이터 내부로 값을 되돌려주지도 않는다. 따라서 타입은 `<T, void, void>` 가 된다.

```typescript
interface Todo {
  userId: number;
  id: number;
  title: string;
  completed: boolean;
}

const slowFetcher = async function* (
  arr: number[],
  fetchIntervalTime: number
): GeneratorFn<Todo> {
  for (let i = 0; i < arr.length; i++) {
    const fetcher = fetch(
      `https://jsonplaceholder.typicode.com/todos/${arr[i]}`
    ).then((res) => {
      if (!res.ok) {
        throw new Error(res.statusText);
      }
      return res.json() as Promise<Todo>;
    });

    // Fetch speed slows down at least fetchIntervalTime.
    const result = await Promise.all([fetcher, delay(fetchIntervalTime)]);

    yield result[0];
  }
};
```

`Todo` 타입은 JsonPlaceholder가 되돌려 주는 JSON의 타입이다. 그냥 `unknown`으로 처리해도 작동에는 문제없지만 타입스크립트의 타입 추론 기능의 도움을 받을 수 없어지므로 가능한 타입 정보를 제공해주는 것이 좋다. T에 Todo 타입을 대입해주었으므로 slowFetcher의 타입 시그니처는 `<Todo, void, void>`가 된다. 마지막으로 `return res.json() as Promise<Todo>;`는 타입 단언문으로, 타입을 단언해주지 않으면 `res.json()`는 `Promise<any>` 타입을 돌려준다.

```typescript
const waitUntil = async function <T>(
  generator: AsyncIterable<T>,
  timeWait: number
) {
  const arr: Awaited<T>[] = [];
  const workerFn = async () => {
    for await (const obj of generator) {
      arr.push(obj);
    }
  };

  await Promise.race([workerFn(), delay(timeWait)]);
  return arr;
};
```

타입스크립트에서 제네레이터는 `Iterable` 또는 `AsyncIterable`타입을 돌려준다. 이 부분은 타입스크립트 컴파일러가 알려준 대로 타입을 지정한거라 자세히는 알지 못한다. 다만 자동으로 타입 추론이 되지는 않았는데, `const arr`의 경우 `any` 타입으로 추론되기 때문에 for await의 obj의 타입을 보고 그대로 넣었다.

실행해보면 `slowEmitter`는 3까지, `slowFetcher`는 4까지 결과를 돌려주는 것을 볼 수 있다. `slowEmitter`의 경우 첫 번째 yield는 지연 없이 yield한다는 사실을 기억할것. 반면 `slowFetcher`는 첫 번째 호출부터 500밀리초의 지연이 걸린다.
