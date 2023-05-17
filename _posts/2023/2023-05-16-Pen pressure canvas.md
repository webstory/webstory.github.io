---
category: 2023
tags:
  [
    "2023",
    "dev",
    "javascript",
    "html5",
    "canvas",
    "tablet",
    "pen_pressure",
    "pressure",
    "onpointermove",
    "pointer_events",
  ]
---

# 펜 압력 감지가 지원되는 HTML5 Canvas sketch pad

[데모 보기](https://codesandbox.io/p/sandbox/nice-zeh-q9ph2k?file=%2Fsrc%2Froutes%2F%2Bpage.svelte%3A1%2C1){:target="\_blank"}{: .btn .btn--info}

브라우저에서 펜 압력 감지가 지원된 것은 비교적 최근의 일이다. [caniuse.com](https://caniuse.com/?search=pointermove){:target="\_blank"} 에 보면 현재 거의 모든 브라우저에서 지원하고 있기는 한데, 내가 기억하기로는 작년까지는 지원이 안 됐었다. 때문에 [aggie.io](https://aggie.io) 에서도 별도 플러그인이나 브라우저 확장을 설치하도록 안내했었는데, 2023년 현재 가 보면 플러그인 설치 안내가 사라져 있다. 이곳에서도 `pointer events`가 지원되길 학수고대하고 있었던 듯 하다.

포인터 이벤트를 사용하면 `pointerType`, `pressure`, `tangentialPressure`, `tiltX`, `tiltY` 값을 얻을 수 있다. 이 중 `tangentialPressure`는 뭔지 모르겠고, `pointerType`은 마우스 사용시 `mouse`, 펜 태블릿 사용시 `pen`으로 표시된다. `pressure`는 펜 압력을 나타내는데, 0~1 사이의 값을 가진다. 마우스의 경우 클릭시 0.5의 값을 가진다. `tiltX`, `tiltY`는 펜의 기울기를 나타내는데, 0~90 사이의 값을 가진다.

위의 데모에서는 원형 브러시를 사용했기 때문에 tilt를 적용하지 못했다.

펜 압력 이벤트에 따라 그림을 그리는 기능을 구현하는데 중점을 두었기 때문에 색상 팔레트 기능이라던지 기타 기능은 최소한만 구현되어 있다. 기반이 된 코드는 [이곳](https://www.youtube.com/shorts/GEJnfwRah-w)에서 참고했다. 유튜브 쇼츠 동영상인데, 이걸 svelte에 맞게 변환한 것이다.

핵심 코드는 아래와 같다.

```typescript
    on:pointermove|preventDefault|stopPropagation|self={(e) => {
      const { pointerType, pressure, tangentialPressure, tiltX, tiltY } = e;
      console.log({ pointerType, pressure, tangentialPressure, tiltX, tiltY });
      if (isPressed && pressure > 0) {
        const x2 = e.offsetX;
        const y2 = e.offsetY;
        drawCircle(x2, y2, pressure);
        drawLine(x, y, x2, y2, pressure);
        x = x2;
        y = y2;
      }
    }}
```

이벤트 핸들러에 `preventDefault`와 `stopPropagation`을 추가했다. 이걸 추가하지 않으면 펜으로 그림을 그릴 때 컨텍스트 메뉴가 나오거나 스크롤이 되거나 해서 그림을 그릴 수가 없다. `pointerdown과` `pointerup`이벤트에도 같은 이유로 이들 modifier가 추가되어 있다. 굳이 svelte event modifier를 사용할 필요는 없고 아래와 같이 해도 된다.

```typescript
on:pointermove={(e) => {
  e.preventDefault();
  e.stopPropagation();
  ...
}}
```
