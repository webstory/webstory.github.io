---
category: 2023
tag: ["2023", "typescript", "effective_typescript", "study"]
author_profile: false
# sidebar:
#   nav: "how-to-github-pages"
# search: false # default true
---

# 이펙티브 타입스크립트 Effective TypeScript

이 글은 한빛출판사의 **이펙티브 타입스크립트**를 공부하면서 내가 이미 아는 내용은 적지 않고 내가 기록하고 싶은 것들만 추려서 정리한 것들이다.

참고 사이트들

- https://dennis-emmental.gitbooks.io/typescript-deep-dive-korean/content/
- https://github.com/roy-jung/effective-typescript
- https://www.youtube.com/playlist?list=PLjQV3hketAJmXGaWCMGB9-085EiefWcyw

대부분의 소스 코드들은 https://github.com/roy-jung/effective-typescript 에서 복사하였다.

## 챕터 1 타입스크립트 알아보기

### 기초 설정

noImplicitAny
strictNullChecks
그냥 strict: true 면 대개는 오케이

### Item 3 코드 생성과 타입이 관계없음을 이해하기

noEmitOnError

객체 타입 정보를 런타임에도 유지하고 싶다면 interface가 아닌 class를 쓰라.
타입 체크는 instanceof class 사용

```typescript
interface Square {
  width: number;
}
interface Rectangle extends Square {
  height: number;
}
type Shape = Square | Rectangle;
function calculateArea(shape: Shape) {
  if ("height" in shape) {
    shape; // Type is Rectangle
    return shape.width * shape.height;
  } else {
    shape; // Type is Square
    return shape.width * shape.width;
  }
}
```

물론 클래스도 type 키워드로 취급할 수 있음

## 챕터 2 타입스크립트의 타입 시스템

### Item 7 타입이 값들의 집합이라고 생각하기

타입 작은 순서대로

- never
- literal

```typescript
type A = "A";
type AB12 = "A" | "B" | 12;
```

- type intersection

```typescript
interface Person { name: string; }
interface Lifespan { birth: Date; death?: Date; }
type PersonSpan = Person & Lifespan // merge
type K = keyof (Person | Lifespan) // == typeof never

keyof(A&B) = (keyof A) | (keyof B);
keyof(A|B) = (keyof A) & (keyof B);
```

extends 키워드는 타입 시스템에서는 상속 개념보다는 '~의 부분집합'의 개념으로 이해하는 게 낫다.

extends 키워드 제네릭 한정사

```typescript
interface Point {
  x: number;
  y: number;
}
type PointKeys = keyof Point; // Type is "x" | "y"

function sortBy<K extends keyof T, T>(vals: T[], key: K): T[] {
  // COMPRESS
  vals.sort((a, b) => (a[key] === b[key] ? 0 : a[key] < b[key] ? -1 : +1));
  return vals; // END
}
const pts: Point[] = [
  { x: 1, y: 1 },
  { x: 2, y: 0 },
];
sortBy(pts, "x"); // OK, 'x' extends 'x'|'y' (aka keyof T)
sortBy(pts, "y"); // OK, 'y' extends 'x'|'y'
sortBy(pts, Math.random() < 0.5 ? "x" : "y"); // OK, 'x'|'y' extends 'x'|'y'
sortBy(pts, "z");
// ~~~ Type '"z"' is not assignable to parameter of type '"x" | "y"
```

### Item 8 타입 공간과 값 공간의 심벌 구분하기

```typescript
class Cylinder {
  radius = 1;
  height = 1;
}

function calculateVolume(shape: unknown) {
  if (shape instanceof Cylinder) {
    shape; // OK, type is Cylinder
    shape.radius; // OK, type is number
  }
}
const v = typeof Cylinder; // Value is "function"
type T = typeof Cylinder; // Type is typeof Cylinder
type C = InstanceType<typeof Cylinder>; // Type is Cylinder
```

클래스에 대해서는 `InstanceType<typeof Cylinder>` 사용

### Item 9 타입 단언보다는 타입 선언을 사용하기

```typescript
interface Person {
  name: string;
}

const alice: Person = { name: "Alice" }; // Type is Person
const bob = { name: "Bob" } as Person; // Type is Person
```

둘이 같아보이겠지만 as Person 은 일종의 강제 타입 캐스팅이다. 이런 경우가 발생한다.

```typescript
interface Person {
  name: string;
}
const alice: Person = {};
// ~~~~~ Property 'name' is missing in type '{}'
//       but required in type 'Person'
const bob = {} as Person; // No error
```

### Item 10 객체 래퍼 타입 피하기

어지간해서는 string을 써야지 String 을 쓰지 말라 뭐 그런 얘기.
이렇게 쓰지 말란거다.

```typescript
const s: String = "primitive";
const n: Number = 12;
const b: Boolean = true;
```

각각 타입 앞글자 소문자로 하는 게 맞다. 래퍼 타입으로 만들면 나중에 함수 등에 넘길 때 시그니처 불일치로 고통받는다.

하지만 new 키워드 없이 사용하는 경우는 기본형을 생성하므로 상관없다.

```typescript
typeof BigInt(1234) === "bigint";
typeof Symbol("sym") === "symbol";
```

### Item 11 잉여 속성 체크의 한계 인지하기

```typescript
interface Room {
  numDoors: number;
  ceilingHeightFt: number;
}
const r: Room = {
  numDoors: 1,
  ceilingHeightFt: 10,
  elephant: "present",
  // ~~~~~~~~~~~~~~~~~~ Object literal may only specify known properties,
  //                    and 'elephant' does not exist in type 'Room'
};
```

그런데

```typescript
interface Room {
  numDoors: number;
  ceilingHeightFt: number;
}
const obj = {
  numDoors: 1,
  ceilingHeightFt: 10,
  elephant: "present",
};
const r: Room = obj; // OK
```

이뿐만 아니라 as 키워드도 문제를 발생시킨다.

```typescript
interface Room {
  numDoors: number;
  ceilingHeightFt: number;
}
function setDarkMode() {}
interface Options {
  title: string;
  darkMode?: boolean;
}
const o = { darkmode: true, title: "Ski Free" } as Options; // OK
```

darkmode 에 오타 있는데 타입 체커를 통과함.

또한 덕 타이핑 때문에 이런 희한한 경우가 생긴다.

```typescript
interface Options {
  title: string;
  darkMode?: boolean;
}
const o1: Options = document; // OK
const o2: Options = new HTMLAnchorElement(); // OK
```

document와 HTMLAnchorElement 둘 다 title 속성을 가짐. darkMode는 원래 옵션 필드. 따라서 OK.

```typescript
interface Options {
  darkMode?: boolean;
  [otherOptions: string]: unknown;
}
const o: Options = { darkmode: true }; // OK
```

darkmode가 otherOptions 카테고리에 들어감.

### Item 12 함수 표현식에 타입 적용하기

```typescript
type BinaryFn = (a: number, b: number) => number;
const add: BinaryFn = (a, b) => a + b;
const sub: BinaryFn = (a, b) => a - b;
const mul: BinaryFn = (a, b) => a * b;
const div: BinaryFn = (a, b) => a / b;
```

시그니처 같으면 중복 작성하지 않기

```typescript
const checkedFetch: typeof fetch = async (input, init) => {
  const response = await fetch(input, init);
  if (!response.ok) {
    throw new Error("Request failed: " + response.status);
  }
  return response;
};
```

```typescript
const checkedFetch: typeof fetch = async (input, init) => {
  //  ~~~~~~~~~~~~   Type 'Promise<Response | HTTPError>'
  //                     is not assignable to type 'Promise<Response>'
  //                   Type 'Response | HTTPError' is not assignable
  //                       to type 'Response'
  const response = await fetch(input, init);
  if (!response.ok) {
    return new Error("Request failed: " + response.status);
  }
  return response;
};
```

throw 해야 할 곳에서 return 한 오류도 잘 찾아내준다.

### Item 13 타입과 인터페이스의 차이점 알기

대개는 섞어서 쓸 수 있고, 타입스크립트 커뮤니티에서는 interface를 더 권장하는 분위기이다.
차이점 1: 유니온 타입은 있지만 유니온 인터페이스는 없다.

```typescript
type AorB = "a" | "b";
```

```typescript
type Input = {
  /* ... */
};
type Output = {
  /* ... */
};
interface VariableMap {
  [name: string]: Input | Output;
}
type NamedVariable = (Input | Output) & { name: string };
```

튜플이나 배열 타입은 type키워드가 낫다

```typescript
type Pair = [number, number];
type StringList = string[];
type NamedNums = [string, ...number[]];
```

이런 타입에 리터럴 할당할 때는 as const 키워드 필요.

반면 인터페이스는 보강 augment 가 가능하다. 선언 병합 declaration merging 이라고도 함

```typescript
interface IState {
  name: string;
  capital: string;
}
interface IState {
  population: number;
}
const wyoming: IState = {
  name: "Wyoming",
  capital: "Cheyenne",
  population: 500_000,
}; // OK
```

별도 타입을 만들어서 type union 하는 것보다는 좀 더 간편하다. 다만 단일 프로젝트 내에서 이러진 말것.

### Item 14 타입 연산과 제네릭 사용으로 반복 줄이기

- 요약: Pick, Partial, ReturnType

```typescript
interface State {
  userId: string;
  pageTitle: string;
  recentFiles: string[];
  pageContents: string;
}
type TopNavState = {
  [k in "userId" | "pageTitle" | "recentFiles"]: State[k];
};
type TopNavState = Pick<State, "userId" | "pageTitle" | "recentFiles">;
```

```typescript
interface SaveAction {
  type: "save";
  // ...
}
interface LoadAction {
  type: "load";
  // ...
}
type Action = SaveAction | LoadAction;
// ❌ type ActionType = 'save' | 'load' // Repeated types!
type ActionType = Action["type"]; // Type is "save" | "load"
type ActionRec = Pick<Action, "type">; // {type: "save" | "load"}
```

업데이트 액션 등에 필요한 OptionalUpdate도 이런 식으로

```typescript
interface Options {
  width: number;
  height: number;
  color: string;
  label: string;
}
type OptionsKeys = keyof Options;
// Type is "width" | "height" | "color" | "label"

type OptionsUpdate = { [k in keyof Options]?: Options[k] };
```

OptionsUpdate는 아래와 같다.

```typescript
interface OptionsUpdate {
  width?: number;
  height?: number;
  color?: string;
  label?: string;
}
```

내장 제네릭 Partial 을 쓰면 더 간단히 된다.

```typescript
class UIWidget {
  constructor(init: Options) {
    /* ... */
  }
  update(options: Partial<Options>) {
    /* ... */
  }
}
```

디폴트값이 있을 때는 typeof로 추론시킬 수 있다.

```typescript
const INIT_OPTIONS = {
  width: 640,
  height: 480,
  color: "#00FF00",
  label: "VGA",
};
type Options = typeof INIT_OPTIONS;
```

물론 필요한 경우 초기값에 타입 명시도 가능.

리턴 타입에 대한 타입 정의

```typescript
function getUserInfo(userId: string) {
  // COMPRESS
  const name = "Bob";
  const age = 12;
  const height = 48;
  const weight = 70;
  const favoriteColor = "blue";
  // END
  return {
    userId,
    name,
    age,
    height,
    weight,
    favoriteColor,
  };
}
// Return type inferred as { userId: string; name: string; age: number, ... }

type UserInfo = ReturnType<typeof getUserInfo>;
```

일종의 타입 템플릿으로 사용도 가능. `<T extends Name>` 부분 볼것.

```typescript
interface Name {
  first: string;
  last: string;
}
type DancingDuo<T extends Name> = [T, T];

const couple1: DancingDuo<Name> = [
  { first: "Fred", last: "Astaire" },
  { first: "Ginger", last: "Rogers" },
]; // OK
const couple2: DancingDuo<{ first: string }> = [
  // ~~~~~~~~~~~~~~~
  // Property 'last' is missing in type
  // '{ first: string; }' but required in type 'Name'
  { first: "Sonny" },
  { first: "Cher" },
];
```

위에서 쓴 Pick 제네릭 말인데,

```typescript
type Pick<T, K> = { [k in K]: T[k] };
// ~ 'K'타입은 'string | number | symbol' 타입에 할당할 수 없습니다.

type Pick<T, K extends keyof T> = { [k in K]: T[k] }; // OK
```

### Item 15 동적 데이터에 인덱스 시그니처 사용하기

`[property: string]: string` 이 인덱스 시그니처다.

```typescript
type Rocket = { [property: string]: string };
const rocket: Rocket = {
  name: "Falcon 9",
  variant: "v1.0",
  thrust: "4,940 kN",
}; // OK
```

근데 이건 문제가 있다.

- 잘못된 키를 포함해 모든 키를 허용함: name 대신 Name 으로 쓰면?
- 특정 키가 필요하지 않음: {} 도 유효한 Rocket
- 키마다 다른 타입을 가질 수 없음: thrust를 number로 하고 싶었다면?
- 타입스크립트 언어 서비스가 도움을 주지 못함

하지만 CSV파서라면? 이 때는 `as unknown as ProductRow[]` 라고 씀.

```typescript
function parseCSV(input: string): { [columnName: string]: string }[] {
  const lines = input.split("\n");
  const [header, ...rows] = lines;
  return rows.map((rowStr) => {
    const row: { [columnName: string]: string } = {};
    rowStr.split(",").forEach((cell, i) => {
      row[header[i]] = cell;
    });
    return row;
  });
}
interface ProductRow {
  productId: string;
  name: string;
  price: string;
}

declare let csvData: string;
const products = parseCSV(csvData) as unknown as ProductRow[];
```

비슷한 필드가 여러개 등장하면 Record타입을 쓸 수 있다.

```typescript
type Vec3D = Record<"x" | "y" | "z", number>;
// Type Vec3D = {
//   x: number;
//   y: number;
//   z: number;
// }
type ABC = { [k in "a" | "b" | "c"]: k extends "b" ? string : number };
// Type ABC = {
//   a: number;
//   b: string;
//   c: number;
// }
```

### Item 16 number 인덱스 시그니처보다는 Array, 튜플, ArrayLike를 사용하기

```typescript
const xs = [1, 2, 3];
const tupleLike: ArrayLike<string> = {
  "0": "A",
  "1": "B",
  length: 2,
}; // OK

function checkedAccess<T>(xs: ArrayLike<T>, i: number): T {
  if (i < xs.length) {
    return xs[i];
  }
  throw new Error(`Attempt to access ${i} which is past end of array.`);
}
```

### Item 17 변경 관련된 오류 방지를 위해 readonly 사용하기

```typescript
function arraySum(arr: readonly number[]) {
  let sum = 0,
    num;
  while ((num = arr.pop()) !== undefined) {
    // ~~~ 'pop' does not exist on type 'readonly number[]'
    sum += num;
  }
  return sum;
}
```

순수 함수 작성할 때는 어지간해서는 readonly 쓰는 게 좋다. 내부에서 객체 수정이 필요하면 전개 연산자 `...` 사용해서 복사본 생성하는 트릭을 쓸 수 있다.

ReadOnly 제네릭도 존재함.

```typescript
interface Outer {
  inner: {
    x: number;
  };
}
const o: Readonly<Outer> = { inner: { x: 0 } };
o.inner = { x: 1 };
// ~~~~ Cannot assign to 'inner' because it is a read-only property
o.inner.x = 1; // OK
```

내부도 변경 불가하게 얼려버리고 싶으면 ts-essentials에 있는 DeepReadonly 를 사용하라.

인덱스 시그니처에도 readonly를 쓸 수 있다. 읽기는 허용하되 쓰기는 방지하는 효과가 있음.

```typescript
let obj: { readonly [k: string]: number } = {};
// Or Readonly<{[k: string]: number}
obj.hi = 45;
//  ~~ Index signature in type ... only permits reading
obj = { ...obj, hi: 12 }; // OK
obj = { ...obj, bye: 34 }; // OK
```

### Item 18 매핑된 타입을 사용하여 값을 동기화하기

```typescript
interface ScatterProps {
  // The data
  xs: number[];
  ys: number[];

  // Display
  xRange: [number, number];
  yRange: [number, number];
  color: string;

  // Events
  onClick: (x: number, y: number, index: number) => void;
}
const REQUIRES_UPDATE: { [k in keyof ScatterProps]: boolean } = {
  xs: true,
  ys: true,
  xRange: true,
  yRange: true,
  color: true,
  onClick: false,
};

function shouldUpdate(oldProps: ScatterProps, newProps: ScatterProps) {
  let k: keyof ScatterProps;
  for (k in oldProps) {
    if (oldProps[k] !== newProps[k] && REQUIRES_UPDATE[k]) {
      return true;
    }
  }
  return false;
}
```

`[k in keyof ScatterProps]`는 타입 체커에게 REQUIRES_UPDATE가 ScatterProps과 동일한 속성을 가져야 한다는 정보를 제공.

## 챕터 3 타입 추론

### Item 19 추론 가능한 타입을 사용해 장황한 코드 방지하기

```typescript
// Don't do this:
app.get("/health", (request: express.Request, response: express.Response) => {
  response.send("OK");
});

// Do this:
app.get("/health", (request, response) => {
  response.send("OK");
});
```

### Item 20 다른 타입에는 다른 변수 사용하기

대충 const 쓰면 해결되는 문제

### Item 21 타입 넓히기

as const 키워드

```typescript
const v1 = {
  x: 1,
  y: 2,
}; // Type is { x: number; y: number; }

const v2 = {
  x: 1 as const,
  y: 2,
}; // Type is { x: 1; y: number; }

const v3 = {
  x: 1,
  y: 2,
} as const; // Type is { readonly x: 1; readonly y: 2; }
```

### Item 22 타입 좁히기

if 문으로 좁히는 것

```typescript
const el = document.getElementById("foo"); // Type is HTMLElement | null
if (!el) throw new Error("Unable to find #foo");
el; // Now type is HTMLElement
```

```typescript
function contains(text: string, search: string | RegExp) {
  if (search instanceof RegExp) {
    search; // Type is RegExp
    return !!search.exec(text);
  }
  search; // Type is string
  return text.includes(search);
}
```

```typescript
function contains(text: string, terms: string | string[]) {
  const termList = Array.isArray(terms) ? terms : [terms];
  termList; // Type is string[]
  // ...
}
```

Tagging 테크닉 사용

```typescript
interface UploadEvent {
  type: "upload";
  filename: string;
  contents: string;
}
interface DownloadEvent {
  type: "download";
  filename: string;
}
type AppEvent = UploadEvent | DownloadEvent;

function handleEvent(e: AppEvent) {
  switch (e.type) {
    case "download":
      e; // Type is DownloadEvent
      break;
    case "upload":
      e; // Type is UploadEvent
      break;
  }
}
```

리터럴 타입을 사용한 트릭이다.

### Item 23 한꺼번에 객체 생성하기

객체에 갱신 작업 하지 말고 전개 연산자 `...` 써서 하라는 얘기.

### Item 24 일관성 있는 별칭 사용하기

### Item 25 비동기 코드에는 콜백 대신 async 함수 사용하기

### Item 26 타입 추론에 문맥이 어떻게 사용되는지 이해하기

### Item 27 함수형 기법과 라이브러리로 타입 흐름 유지하기

lodash 찬양?

## 챕터 4 타입 설계

### Item 28 유효한 상태만 표현하는 타입을 지향하기

```typescript
interface State {
  pageText: string;
  isLoading: boolean;
  error?: string;
}
```

이러면 `{ isLoading: true, error: 'Error' }`인 이상한 상태까지 허용해버린다.

```typescript
interface RequestPending {
  state: "pending";
}
interface RequestError {
  state: "error";
  error: string;
}
interface RequestSuccess {
  state: "ok";
  pageText: string;
}
type RequestState = RequestPending | RequestError | RequestSuccess;

interface State {
  currentPage: string;
  requests: { [page: string]: RequestState };
}
```

이렇게 하라.

### Item 29 사용할 때는 너그럽게, 생성할 때는 엄격하게

### Item 30 문서에 타입 정보를 쓰지 않기

주석질 하지 말란거다. 타입스크립트가 알아서 하게 둔다. 대신 `/** */` 으로 함수나 필드 등에 '사람에게 도움되는' 부가정보를 줄 순 있다.

### Item 31 타입 주변에 null값 배치하기

나쁜 예

```typescript
function extent(nums: number[]) {
  let min, max;
  for (const num of nums) {
    if (!min) {
      min = num;
      max = num;
    } else {
      min = Math.min(min, num);
      max = Math.max(max, num);
    }
  }
  return [min, max];
}
```

좋은 예

```typescript
function extent(nums: number[]) {
  let result: [number, number] | null = null;
  for (const num of nums) {
    if (!result) {
      result = [num, num];
    } else {
      result = [Math.min(num, result[0]), Math.max(num, result[1])];
    }
  }
  return result;
}
```

사실 이런 종류의 문제는 다 null때문에 발생하는 것으로, 기본값 또는 널 객체를 사용하면 이 문제를 근본적으로 해결할 수 있다.

### Item 32 유니온의 인터페이스보다는 인터페이스의 유니온을 사용하기

나쁜 예

```typescript
interface Layer {
  layout: FillLayout | LineLayout | PointLayout;
  paint: FillPaint | LinePaint | PointPaint;
}
```

layout은 LineLayout인데 paint가 PointPaint인 이상한 레이어도 허용돼버린다.

좋은 예

```typescript
interface FillLayer {
  type: "fill";
  layout: FillLayout;
  paint: FillPaint;
}
interface LineLayer {
  type: "line";
  layout: LineLayout;
  paint: LinePaint;
}
interface PointLayer {
  type: "paint";
  layout: PointLayout;
  paint: PointPaint;
}
type Layer = FillLayer | LineLayer | PointLayer;
```

Layer 타입의 Layer.type은 자동으로 'fill' | 'line' | 'paint' 로 추론된다. 따로 인터페이스 Layer를 만들 필요 없다.

### Item 33 string 타입보다 더 구체적인 타입 사용하기

```typescript
function pluck<T>(record: T[], key: keyof T): T[keyof T][] {
  return record.map((r) => r[key]);
}
```

그런데 key값으로 하나를 뽑아보면 객체 내의 모든 프로퍼티들의 타입을 다 뭉개놓은(union) 것을 볼 수 있을 것이다. 따라서,

```typescript
function pluck<T, K extends keyof T>(record: T[], key: K): T[K][] {
  return record.map((r) => r[key]);
}
```

이렇게 하라. 사실 lodash의 pluck 쓰면 됨.

### Item 34 부정확한 타입보다는 미완성 타입을 사용하기

### Item 35 데이터가 아닌, API와 명세를 보고 타입 만들기

### Item 36 해당 분야의 용어로 타입 이름 짓기

설계 관련 격언 같은 것.

### Item 37 공식 명칭에는 상표를 붙이기

라이브러리 개발자나 신경쓸 이야기

## 챕터 5 any 다루기

### Item 38 any타입은 가능한 한 좁은 범위에서만 사용하기

사실 unknown 쓰는 게 낫다. 이것도 가능한 좁은 범위에서 써야 한다.

### Item 39 any를 구체적으로 변형해서 사용하기

### Item 40 함수 안으로 타입 단언문 감추기

### Item 41 any의 진화를 이해하기

any로 했더라도 타입스크립트 추론 엔진은 해당 타입이 if를 통과할 때마다 타입을 좁히려고 시도한다.

### Item 42 모르는 타입의 값에는 any대신 unknown을 사용하기

### Item 43 몽키 패치보다는 안전한 타입을 사용하기

### Item 44 타입 커버리지를 추적하여 타입 안전성 유지하기

## 챕터 6 타입 선언과 @types

### Item 45 devDependencies에 typescript와 @types 추가하기

### Item 46 타입 선언과 관련된 세 가지 버전 이해하기

- 라이브러리의 버전
- 타입 선언(@types)의 버전
- 타입스크립트의 버전

### Item 47 공개 API에 등장하는 모든 타입을 익스포트하기

### Item 48 API주석에 TSDoc 사용하기

```typescript
interface Vector3D {}
/** A measurement performed at a time and place. */
interface Measurement {
  /** Where was the measurement made? */
  position: Vector3D;
  /** When was the measurement made? In seconds since epoch. */
  time: number;
  /** Observed momentum */
  momentum: Vector3D;
}

/**
 * Generate a greeting.
 * @param name Name of the person to greet
 * @param salutation The person's title
 * @returns A greeting formatted for human consumption.
 */
function greetFullTSDoc(name: string, title: string) {
  return `Hello ${title} ${name}`;
}
```

주석에 마크다운 문법 사용 가능.

### Item 49 콜백에서 this에 대한 타입 제공하기

그냥 화살표 함수 써라.

```typescript
declare function makeButton(props: { text: string; onClick: () => void }): void;
class ResetButton {
  render() {
    return makeButton({ text: "Reset", onClick: this.onClick });
  }
  onClick = () => {
    alert(`Reset ${this}`); // "this" always refers to the ResetButton instance.
  };
}
```

또는 Object.call을 사용할 수 있다.

```typescript
function addKeyListener(
  el: HTMLElement,
  fn: (this: HTMLElement, e: KeyboardEvent) => void
) {
  el.addEventListener("keydown", (e) => {
    fn.call(el, e);
  });
}
```

정 헷갈리면 이벤트 리스너에는 function 키워드 사용한다.

```typescript
declare let el: HTMLElement;
addKeyListener(el, function (e) {
  this.innerHTML; // OK, "this" has type of HTMLElement
});
```

### Item 50 오버로딩 타입보다는 조건부 타입을 사용하기

```typescript
function double<T extends number | string>(
  x: T
): T extends string ? string : number;
function double(x: any) {
  return x + x;
}
const num = double(12); // number
const str = double("x"); // string

// function f(x: string | number): string | number
function f(x: number | string) {
  return double(x);
}
```

### Item 51 의존성 분리를 위해 미러 타입 사용하기

nodejs하고 브라우저 환경하고 좀 다른데 공통 라이브러리를 작성하고 싶은 경우. 대표적으로 @types/node 에만 존재하는 Buffer

```typescript
interface CsvBuffer {
  toString(encoding: string): string;
}
function parseCSV(
  contents: string | CsvBuffer
): { [column: string]: string }[] {
  // COMPRESS
  return [];
  // END
}

parseCSV(new Buffer("column1,column2\nval1,val2", "utf-8")); // OK
```

CSVBuffer는 Buffer타입을 '흉내낸다.' (Duck typing)

### Item 52 테스팅 타입의 함정에 주의하기

assertType 이란 게 있고, 근데 이것도 통과하는 이상한 경우가 있고, 제대로 된 assertType사용 방법은 Parameter와 ReturnType을 분리하여 테스트하는 것
근데 이건 잘 이해가 안 된다.

## 챕터 7 코드를 작성하고 실행하기

### Item 53 타입스크립트 기능보다는 ECMAScript 기능을 사용하기

### Item 54 객체를 순회하는 노하우

### Item 55 DOM 계층 구조 이해하기

일단 타입스크립트는 DOM에 대해 모른다. 이 정보른 런타임에서 자바스크립트 엔진만 안다.

```typescript
function addDragHandler(el: HTMLElement) {
  el.addEventListener("mousedown", (eDown) => {
    const dragStart = [eDown.clientX, eDown.clientY];
    const handleUp = (eUp: MouseEvent) => {
      el.classList.remove("dragging");
      el.removeEventListener("mouseup", handleUp);
      const dragEnd = [eUp.clientX, eUp.clientY];
      console.log(
        "dx, dy = ",
        [0, 1].map((i) => dragEnd[i] - dragStart[i])
      );
    };
    el.addEventListener("mouseup", handleUp);
  });
}
```

### Item 56 정보를 감추는 목적으로 private 사용하지 않기

정 쓰려면 클로저 사용

```typescript
declare function hash(text: string): number;

class PasswordChecker_ {
  checkPassword: (password: string) => boolean;
  constructor(passwordHash: number) {
    this.checkPassword = (password: string) => {
      return hash(password) === passwordHash;
    };
  }
}

const checker = new PasswordChecker(hash("s3cret"));
checker.checkPassword("s3cret"); // Returns true
```

또는, ECMAScript 에 도입된 비공개 필드 사용
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_class_fields 이건데 private 키워드도 사용 가능한듯?
https://stackoverflow.com/questions/59641564/what-are-the-differences-between-the-private-keyword-and-private-fields-in-types 에 따르면, 아니랜다.

```typescript
declare function hash(text: string): number;
class PasswordChecker {
  private password: string;

  constructor() {
    this.password = "s3cret";
  }

  checkPassword(password: string) {
    return password === this.password;
  }
}

const checker = new PasswordChecker();
const password = (checker as any).password;
```

자잘하게 신경쓰기 싫다. 어차피 런타임에서 메모리 까면 private 필드 내용 볼 수 있다.

### Item 57 소스맵을 사용하여 타입스크립트 디버깅하기

## 챕터 8 타입스크립트로 마이그레이션하기

### Item 58 모던 자바스크립트로 작성하기

### Item 59 타입스크립트 도입 전에 @ts-check와 JSDoc으로 시험해 보기

### Item 60 allowJs로 타입스크립트와 자바스크립트 같이 사용하기

### Item 61 의존성 관계에 따라 모듈 단위로 전환하기

### Item 62 마이그레이션의 완성을 위해 noImplicitAny 설정하기
