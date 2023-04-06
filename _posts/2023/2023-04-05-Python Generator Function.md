---
category: 2023
tags: ["2023", "dev", "python", "generator"]
---

# 파이썬 제네레이터 함수

파이썬의 제네레이터 함수는 이터레이터 Iterator 를 반환하는 함수로 yield 키워드를 사용하여 값을 반환한다. 제네레이터 함수는 파이썬 2.2 버전부터 도입되었으며(따라서 파이썬 3에서는 버전 관계없이 사용할 수 있다) 함수 선언시 특별히 제네레이터라는 표시를 붙이지는 않는다. 자바스크립트 함수는 별도로 function\* 과 같은 제네레이터 함수 선언을 해 주어야 하는 것과는 조금 차이가 있다.

제네레이터 함수는 함수를 리턴하지 않고 호출측으로 값을 반환할 수 있는 함수라고 이해하면 쉽다. 즉 함수 내부에 있는 지역 변수들의 상태를 **기억**하면서 호출 측으로 값을 전달해줄 수 있는 함수이다. 예를 들어보자.

```python
def generatorFn():
  yield 1
  yield 2
  yield 3
  yield 4
  yield 5
  yield 10

if __name__ == '__main__':
  for n in generatorFn():
    print(n)
```

결과:

```
1
2
3
4
5
10
```

여기까지는 그다지 흥미롭지 않다. yield 대신에 그냥 배열을 반환하면 되니까.
하지만 만약 무한을 반환해야 한다면 어떨까?

```python
def numseq():
  x = 0
  while True:
    x = x + 1
    yield x


if __name__ == '__main__':
  gen = numseq()
  next(gen)
  next(gen)
  for n in range(3):
    print(next(gen))

  print("==========")
  for n in range(3):
    print(next(gen))
```

결과

```
3
4
5
==========
6
7
8
```

`next(gen)` 에 의해 `[1, 2]`는 건너뛰고 첫 루프에서 `[3, 4, 5]` 그리고 두 번째 루프에서 `[6, 7, 8]`을 출력한 것을 볼 수 있다. numseq 제네레이터는 일종의 "자연수 전체의 집합"과 같이 작동했고 그걸 사용하는 측에서 하나씩 수열을 뽑아내 사용한 것을 볼 수 있다.

무한스크롤 등을 구현할 때 이 제네레이터 함수를 사용하면 마지막까지 읽은 포스트의 ID를 호출 측에서 별도로 기억하지 않고 제네레이터에게 연속된 포스트의 출력 책임을 위임할 수 있다.

그런데 가끔은 제네레이터의 출력을 제어하고 싶을 때가 있을 것이다. 대표적인 경우가 "초기화"일텐데, 대개는 새로운 제네레이터를 만드는 것으로 충분하지만 제네레이터 함수로 값을 "밀어넣어"주어서 제네레이터 함수가 스스로를 조절하게 만들 수도 있다.

이때 gen.send() 함수를 사용할 수 있다. 여기서 gen은 제네레이터의 인스턴스이다.

```python
def doubler():
  x = 0
  while True:
    y = yield x * 2
    if y == None:
      x = x * 2
    else:
      x = y


if __name__ == '__main__':
  gen = doubler()
  next(gen) # 제네레이터 시작(반드시 이런 식으로 시동 걸어줘야 함)
  print(gen.send(1)) # 2
  print(next(gen)) # 4
  print(next(gen)) # 8
  print(next(gen)) # 16
  print(next(gen)) # 32
  print(gen.send(6)) # 12
  print(gen.send(5)) # 10
```

## yield from

필요에 따라서 generator 안에서 새로운 generator를 불러야 할 때도 있다. 또는 일반 리스트를 제네레이터 객체로 변경해야 할 수도 있다. 이때 `yield from`을 사용한다.

```python
def run(n):
  yield from [n, n + 1, n + 2]

for n in run(1):
  print(n) # 1, 2, 3
```

더 많은 응용 사례는 [Generator Yield From](https://python.astrotech.io/advanced/generator/yield-from.html){:target="\_blank"} 을 참고하라.
