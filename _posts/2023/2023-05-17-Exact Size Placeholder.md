---
category: 2023
tags:
  [
    "2023",
    "dev",
    "javascript",
    "svelte",
    "express",
    "lazy_load",
    "placeholder",
    "image",
    "head",
    "prefetch",
  ]
---

# 정확한 크기의 placeholder를 갖는 이미지 리스트 만들기

정말 ChatGPT의 도움이 없었다면 몇 날 며칠을 헤맸을지도 모를 문제다. 다만 ChatGPT도 한번에 답을 안 주고 내가 몇 번을 묻고 또 물어보니까 그제야 정답을 알려주더라. 어째 본인(?)도 몰랐었는데 내가 하도 꼬치꼬치 캐물으니까 그제서야 공식 매뉴얼을 읽고 와서 정답을 알려준 느낌? 저번에 소수 탐색 알고리즘에서도 부등호 하나가 잘못돼있었는데 그걸 잡아내지 못하고 "코드에 문제없어요." 라고 대답했었는데, 이 ChatGPT란 놈, 의외로 허당끼가 있다. 코드를 엄밀하게 분석하지 않고 진짜 사람이 보는 것처럼 코드를 읽고 해석한다! 앞으로 얘하고 대화할 때 주의해야겠다.

이번 거는 전체 코드를 올리지는 않는다. 전체를 올리기엔 양이 너무 많다. 기본적인 `SvelteKit` 설정 방법과 `express.js` 설정 방법은 이미 잘 알고 있다고 가정하고 포스팅하겠다.

조적 레이아웃(Masonly layout)이나 무한스크롤을 구현하다보면 이미지의 세로 길이를 모르기 때문에 일단 대충 맞추고 나중에 이미지가 로드되면 그 때 이미지의 세로 길이를 알아내서 레이아웃을 다시 잡는다. 이 때문에 이미지가 로드되면서 스크롤바가 그야말로 춤을 추는 문제가 발생한다. 물론 클라이언트는 이미지의 세로 길이는커녕 가로 길이도 미리 알 방법이 없다. 하지만 서버는 알 수 있다! 따라서 이미지를 로드하기 전에 미리 이미지의 메타데이터만을 서버에 요청한다면 이미지의 실제 크기를 미리 알아내서 레이아웃을 잡을 수 있을 것이다.

먼저, 간단한 조적 레이아웃을 svelte로 만들었다. 조적 레이아웃이라기보단 그냥 세로 방향 이미지 리본이다.

## 클라이언트

`/src/lib/components/Picture.svelte`

```svelte
<script lang="ts">
  import { onMount } from 'svelte';

  export let src: string;
  export let alt: string | null = null;
  export let width: string = '0px';
  export let height: string = '0px';

  let imgEl: HTMLImageElement;
  let imageLoaded = false;

  $: altText = alt || src;

  onMount(async () => {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            console.log(`Image ${src} is in the viewport!`);
            imgEl.src = src;
            observer.unobserve(imgEl);
          }
        });
      },
      { threshold: 0.1 }
    );

    fetch(src, {
      method: 'HEAD',
    }).then((res) => {
      console.log(res.headers);
      if (res.ok && imageLoaded === false) {
        console.log(`HEAD X-Width:${res.headers.get('X-Width')}, X-Height:${res.headers.get('X-Height')}`);
        if (res.headers.get('X-Width') && res.headers.get('X-Height')) {
          width = res.headers.get('X-Width') + 'px';
          height = res.headers.get('X-Height') + 'px';
        }
      } else {
        console.error(res);
      }
    });

    observer.observe(imgEl);

    return () => {
      observer.disconnect();
    };
  });

  function afterLoad() {
    imageLoaded = true;
    width = imgEl.naturalWidth + 'px';
    height = imgEl.naturalHeight + 'px';
    console.log(`Image ${src} loaded! with size ${width}x${height}`);
  }
</script>

<div class="wrapper" style:width style:height>
  <img bind:this={imgEl} data-src={src} alt={altText} {...$$restProps} on:load|once={afterLoad} />
  {#if !imageLoaded}
    <div class="placeholder" />
  {/if}
</div>

<style>
  .wrapper {
    position: relative;
    overflow: hidden;
  }

  .wrapper * {
    box-sizing: border-box;
  }

  .wrapper img {
    position: relative;
    top: 0;
    left: 0;
  }

  .placeholder {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #eee;
    border: 2px solid #ddd;
    border-radius: 0.25rem;
  }
</style>

```

`/src/routes/+page.svelte`

```svelte
<script lang="ts">
  import Picture from '$lib/components/Picture.svelte';

  const heights = [200, 300, 400, 500, 600, 700, 800, 900, 1000];

  let arr = new Array(30).fill().map((_, i) => {
    return {
      index: i,
      height: heights[Math.floor(Math.random() * heights.length)],
    };
  });
</script>

{#each arr as item}
  <Picture src="http://localhost:3000/200/{item.height}" />
{/each}

```

`Picture.svelte` 에서 이 부분에 주목해주길 바란다.

```typescript
fetch(src, {
  method: "HEAD",
}).then((res) => {
  console.log(res.headers);
  if (res.ok && imageLoaded === false) {
    console.log(
      `HEAD X-Width:${res.headers.get("X-Width")}, X-Height:${res.headers.get(
        "X-Height"
      )}`
    );
    if (res.headers.get("X-Width") && res.headers.get("X-Height")) {
      width = res.headers.get("X-Width") + "px";
      height = res.headers.get("X-Height") + "px";
    }
  } else {
    console.error(res);
  }
});
```

서버에 `HEAD` 요청으로 이미지의 Width와 Height를 알아낸다. 하지만 이 `HEAD`요청은 커스텀 요청으로, 일반적인 웹 서버에서 이미지 에셋에 대해 HEAD요청을 한들 아무런 응답을 주지 않는다. 따라서 위의 Picture컴포넌트는 그것에 대한 fallback을 가지고 있는데, `afterLoad` 함수에서 이미지가 로드된 뒤에 이미지의 실제 width, height를 재설정하는 것을 볼 수 있다.

또한 일반적으로 `IntersectionObserver`는 이것보다 더 상위 컴포넌트 예를 들어 `Masonly.svelte`같은 곳에서 한꺼번에 관리하고 Picture컴포넌트는 Masonly에게 img src의 제어를 위임받아 처리하는 게 더 알맞다. 여기서는 예제를 간단히 하기 위해 개별 Picture마다 IntersectionObserver를 생성했다. 실제로는 이렇게 하면 퍼포먼스에 심각한 영향을 미치므로 이렇게 해서는 안 된다.

## 서버

먼저 서버의 전체 코드를 보자.

```typescript
import express from "express";
import fetch from "node-fetch";
import cors from "cors";

const app = express();
const port = 3000;
app.use(
  cors({
    origin: "*",
    methods: ["GET", "HEAD", "OPTIONS"],
    allowedHeaders: ["Content-Type", "X-Width", "X-Height"],
  })
);
app.use((req, _res, next) => {
  console.log(
    `[${new Date().toISOString()}] ${req.method} ${req.url} Query Parameters:`,
    req.query
  );
  next();
});

// this head handler must be placed before get handler
app.head("/:width/:height", async (req, res) => {
  res.setHeader("Content-Type", "image/jpeg");
  res.setHeader("X-Width", req.params.width);
  res.setHeader("X-Height", req.params.height);
  res.setHeader("Access-Control-Expose-Headers", "X-Width, X-Height");
  res.setHeader(
    "Cache-Control",
    "no-store, no-cache, must-revalidate, proxy-revalidate"
  );
  res.setHeader("Expires", "0");
  res.setHeader("Pragma", "no-cache");
  console.log(`Head image ${req.params.width}x${req.params.height}`);
  res.status(200).end();
});

app.get("/:width/:height", async (req, res) => {
  try {
    const width = req.params.width;
    const height = req.params.height;
    const response = await fetch(`https://picsum.photos/${width}/${height}`);

    if (!response.ok) {
      res.sendStatus(404);
      return;
    }

    // Header must be same as in HEAD request
    res.setHeader("Content-Type", "image/jpeg");
    res.setHeader("X-Width", req.params.width);
    res.setHeader("X-Height", req.params.height);
    res.setHeader("Access-Control-Expose-Headers", "X-Width, X-Height");
    res.setHeader(
      "Cache-Control",
      "no-store, no-cache, must-revalidate, proxy-revalidate"
    );
    res.setHeader("Expires", "0");
    res.setHeader("Pragma", "no-cache");

    response.body!.pipe(res);
  } catch (error) {
    console.error(error);
    res.sendStatus(500);
  }
});

app.listen(port, () => {
  console.log(`Express.js server running on port ${port}`);
});
```

서버 측에서도 이미지의 메타데이터를 미리 계산해 둘 필요는 있다. 예를 들어 redis 같은 걸로 이미지의 width, height 값을 캐싱해두는 작업이 필요하다. ChatGPT는 `sharp` 라이브러리를 쓰던데, 어쨌든 이 블로그에서는 예제를 간단히 하기 위해 `Lorem Picsum`을 사용했다.

`Lorem Picsum`은 파라미터로 width/height 를 주면 그에 맞는 이미지를 되돌려준다. 이 서버에서는 `Loram Picsum`을 프록시한다. 이미지의 width/height를 캐싱해두는 로직을 없애고 파라미터에서 바로 width/height를 뽑아 돌려주므로써 예제를 간단히 하기 위함이다. 실제 응용에서는 이미지의 크기를 계산한 캐시 데이터베이스를 사용해야 할 것이다.

```typescript
import fetch from "node-fetch";
```

원래 nodejs에도 fetch가 있는데 이게 `pipe` 메소드 대신 `pipeThrough`를 사용하고 기타 몇 가지 자잘한 이슈가 있어서 `node-fetch`를 사용했다. `node-fetch` 사용시 기본 `fetch`를 덮어쓰므로 주의해야 한다.

```typescript
import cors from "cors";

const app = express();
const port = 3000;
app.use(
  cors({
    origin: "*",
    methods: ["GET", "HEAD", "OPTIONS"],
    allowedHeaders: ["Content-Type", "X-Width", "X-Height"],
  })
);
```

CORS 헤더를 설정해준다. 기본적으로 `Content-Type`은 허용이 돼 있는데 커스텀 헤더는 따로 설정해 주지 않으면 클라이언트에 전달되지 않고 사라지므로 여기서 `allowHeaders`옵션으로 설정해 주어야 한다. 또한 `HEAD` 메소드도 기본적으로 허용돼있지 않기 때문에 이것도 추가해준다. 서버가 `POST`, `PUT`, `DELETE` 같은 메소드도 처리할 예정이면 그것도 추가해주어야 한다.

```typescript
// this head handler must be placed before get handler
app.head("/:width/:height", async (req, res) => {
  res.setHeader("Content-Type", "image/jpeg");
  res.setHeader("X-Width", req.params.width);
  res.setHeader("X-Height", req.params.height);
  res.setHeader("Access-Control-Expose-Headers", "X-Width, X-Height");
  res.setHeader(
    "Cache-Control",
    "no-store, no-cache, must-revalidate, proxy-revalidate"
  );
  res.setHeader("Expires", "0");
  res.setHeader("Pragma", "no-cache");
  console.log(`Head image ${req.params.width}x${req.params.height}`);
  res.status(200).end();
});
```

이 메소드는 반드시 `app.get`보다 먼저 와야 한다. ChatGPT가 말하길

```
According to the Express.js documentation, if there is no route handler registered specifically for a HEAD request, Express will call the GET route handler (if it exists) for the same path, and then automatically remove the response body before sending the response. This is done to comply with the HTTP/1.1 specification, which states that the HEAD method should return the same headers as a GET request would, but with no response body.

In your case, when the app.get() route handler is defined before the app.head() route handler, Express calls the GET handler for HEAD requests because it hasn't encountered the HEAD handler yet. When you define the app.head() route handler before the app.get() route handler, Express finds the HEAD handler first and correctly executes it for HEAD requests.

To ensure that your server correctly handles HEAD and GET requests separately, make sure to define the app.head() route handler before the app.get() route handler, as you have observed.
```

즉 `get`이 먼저 있으면 `head`대신 `get`을 실행하고 `head`핸들러가 무시된다.

`Access-Control-Expose-Headers` 헤더는 클라이언트에서 접근할 수 있는 헤더를 설정한다. `X-Width`, `X-Height` 헤더를 클라이언트에서 접근할 수 있도록 설정해준다.

나머지 `Cache-Control`이나 그 아래 것들은 캐시 비활성화 관련한 설정으로, 실제 서비스에서는 필요 없다.

마지막으로,

```typescript
res.status(200).end();
```

이걸 이렇게 안 하고 `res.sendStatus(200)`으로 하면 기껏 설정해 놓은 헤더들을 싹 지우고 기본 헤더만 보내버린다. 반드시 `res.status(200).end()`로 마무리해야 한다.

`get`핸들러는 평범한 이미지 서버 프록시이므로 설명을 생략한다. 다만 HTTP1.1 스펙에 `HEAD`요청과 `GET`요청에서 되돌려주는 헤더는 똑같아야 한다고 정의돼있다고 하니까 그 부분 주의해준다.

## 다듬을 것들

이 코드 역시 클라이언트 측에서 다량의 HEAD요청을 보내므로 최적화된 코드라고 볼 수는 없다. 일정 구간별로 이미지 목록을 작성하고 그걸 한꺼번에 서버에 요청하는 편이 통신 부담을 덜 수 있을 것이다. 이미지 뿐만 아니라 컨테이너의 가로폭/세로폭이 정해진 다른 에셋들(동영상 등)에도 적용할 수 있는 방법으로 동영상의 경우에는 `ffprobe`를 사용할 수 있다.

이미지 자체는 지연 로딩을 하지만 이미지의 메타데이터는 페이지 로드시 한꺼번에 전부 요청하므로 이것도 주의해야 한다. 즉 무한스크롤 등에 잘못 사용하면 수백만 개가 넘는 이미지의 메타데이터를 한꺼번에 로드하려고 들다가 서버하고 같이 사이좋게 죽을 수도 있다. 적절한 pagenation은 필수이다.

그리고, 굳이 HEAD요청이 아니어도 된다. 이미지를 페이지 단위로 불러오는 곳에서는 `/images/{:page}/list` 같은 별도의 엔드포인트에서 GET요청을 통해 이미지의 메타데이터를 받아올 수도 있다. 다만 똑같은 URL에서 메소드만 바꾸는 이 방식에 비해 범용성은 좀 떨어질 것이다.
