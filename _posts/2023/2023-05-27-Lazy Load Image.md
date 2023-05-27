---
category: 2023
tags: ["2023", "dev", "svelte", "javascript", "typescript", "image", "lazy"]
---

# Lazy Load Image

개인적인 필요로 Lazy load image svelte component를 만들었다. 이 컴포넌트에는 두 가지 요구사항이 있었는데

1. 뷰포트에 이미지 컴포넌트가 들어올 때까지 이미지의 로드를 보류한다.
2. 여러 URL중 첫 번째로 로드되는 이미지를 로드한다.

## 첫 번째 방법: picture 태그 사용

사실 첫 번째 요구사항보다 두 번째 요구사항을 먼저 구현했었다. 이를 위해 picture 태그를 사용했다.

```html
<picture>
  <source srcset="{srcset[0]}" />
  <source srcset="{srcset[1]}" />
  <img src="{fallback}" alt="{alt}" />
</picture>
```

그런데 이 방법에는 문제가 있었다. 첫 번째 srcset이 404에러를 반환할 경우 picture태그가 다음 이미지를 로드하질 않고 그냥 로드를 실패한 것이다. 원래 source 리스트를 제공하는 목적이 이미지 로드를 실패했을 경우 다음 source를 시도하는 게 아니었던가? 하여튼 브라우저에서는 `NS_BINDING_ABORTED` 오류를 내면서 실패했다.

## 두 번째 방법: fetch로 이미지 미리 로드하기(실패)

참 고약한데, `<img>` 태그는 CORS를 우회하지만 `fetch`는 그렇지 않다. fetch get 뿐만 아니라 fetch head도 실패했다. 물론 서버 측에서 CORS를 허용한다면 이 방법을 쓸 수 있었겠지만, 안타깝게도 내가 로드하려는 이미지는 s3를 통해 제공하고 있었고 `SignedURL`로 접근해야 했다. 그래서 이 방법은 실패했다.

## 세 번째 방법: IntersectionObserver 사용

사실상 네 번째 방법이다. 내가 시도했던 세 번째 방법은 자바스크립트로 img src를 바꿔가면서 로드에 성공하면 루프를 종료하는 것이었다. 그런데 이미 자바스크립트에 손 댄 이상 IntersectionObserver도 같이 사용하는 게 좋을 것 같아서 최종적으로는 이렇게 만들었다.

```html
<script lang="ts">
  import { onMount, onDestroy } from "svelte";

  let imgEl: HTMLImageElement;
  let src = "";
  export let alt = "";

  export let srcset = [];

  let observer;

  onMount(() => {
    if (srcset.length === 0) {
      return;
    }

    observer = new IntersectionObserver(
      async (entries) => {
        if (entries[0].isIntersecting) {
          observer.unobserve(imgEl);
          const img = new Image();
          img.onload = () => {
            src = img.src;
          };
          img.onerror = () => {
            if (srcset.length === 0) {
              return;
            }
            img.src = srcset.shift();
          };

          img.src = srcset.shift();
        }
      },
      {
        root: null,
        rootMargin: "0px",
        threshold: 0.1,
      }
    );
    observer.observe(imgEl);
  });

  onDestroy(() => {
    if (observer) {
      observer.unobserve(imgEl);
    }
  });
</script>

<img bind:this="{imgEl}" {src} {alt} />

<style>
  img {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }
</style>
```

이미지가 뷰포트에 등장하면 srcset을 순서대로 꺼내면서 로드를 시도한다. 코드 상에는 명시적인 루프가 없는데 `onerror`는 `continue`, `onload`는 `break`라고 생각하면 간편할 것이다.

하지만 브라우저 상에서 콘솔 로그에 `404 Not Found`에러가 폭주하는 현상은 막을 수가 없다. 결국 서버 측에서 URL이 로드 가능한지 확인 후 로드 가능한 URL만을 클라이언트로 넘겨주게 만들었는데, 이 과정에서 처음의 목적은 상실했지만 Lazy Load기능은 살아남았고, 이렇게 블로그로 정리한다.

참고로 서버 측에서는 CORS와 상관없이 자유롭게 URL을 검사할 수 있다.
