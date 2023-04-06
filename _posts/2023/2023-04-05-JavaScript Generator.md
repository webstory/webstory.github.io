---
category: 2023
tags: ["2023", "dev", "javascript", "generator"]
---

# 자바스크립트 제네레이터 함수

저번 [파이썬 제네레이터](/2023/Python-Generator-Function) 에서 제네레이터를 다뤄봤는데, 자바스크립트에도 `function*` 이라는 자바스크립트 제네레이터 함수가 제공된다. 개념은 파이썬과 거의 같다.

[MDN Generator](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Generator){:target="\_blank"} 에서 예제를 가져왔다.

```javascript
function* generator() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = generator(); // "Generator { }"

console.log(gen.next().value); // 1
console.log(gen.next().value); // 2
console.log(gen.next().value); // 3
```

하지만 자바스크립트에서 제네레이터 객체를 쓸 때는 대개 비동기 api call 의 반환값을 순서대로 건네주는 것을 이용하려는 때이다. 이때는 [AsyncGenerator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncGenerator){:target="\_blank"}를 사용해야 하는데, 사용법이 조금 다르다.

```javascript
async function* foo() {
  yield await Promise.resolve("a");
  yield await Promise.resolve("b");
  yield await Promise.resolve("c");
}

let str = "";

async function generate() {
  for await (const val of foo()) {
    str = str + val;
  }
  console.log(str);
}

generate();
// Expected output: "abc"
```

함수 선언시 `async function*` 를 사용하고 for loop를 돌 때도 `for await`를 사용한다.
아래는 내가 작성한 크롤러에서 해당 제네레이터를 사용하는 부분을 떼 온 것이다.

```javascript
async function* fetchSubmissionList(token, searchParams) {
  const { sid, user_id } = token;
  let res;

  const defaultSearchParams = {
    sid: sid,
    output_mode: "json",
    submission_ids_only: "yes",
    submissions_per_page: 100,
  };

  // Fetch the first page(mode 1)
  res = await axios.get(siteUrl + "/api_search.php", {
    headers: {},
    params: {
      ...defaultSearchParams,
      ...searchParams,
      get_rid: "yes",
    },
  });

  let { rid, page, pages_count, results_count_all, submissions } = res.data;

  console.log({ pages_count, results_count_all });

  for (const submission of submissions) {
    yield submission.submission_id;
  }

  while (page <= pages_count) {
    page++;

    // Countinuous search(mode 2)
    res = await axios.get(siteUrl + "/api_search.php", {
      headers: {},
      params: {
        ...defaultSearchParams,
        rid: rid,
        page: page,
      },
    });

    let { submissions } = res.data;
    for (const submission of submissions) {
      yield submission.submission_id;
    }
  }
}
```

해당 사이트의 검색 API가 첫 번째 페이지와 두 번째 이후 페이지 검색에 서로 다른 파라메터를 사용하는데 다음 페이지 검색때는 첫 번째 페이지 검색에서 반환하는 `rid`값을 사용하게 되어 있다. 문제는 서로 별개의 키워드를 가진 검색을 여러 개 했을 때 각각의 키워드에 대한 `rid`값을 일일이 기억해야 한다는 것이었다. 즉 1페이지는 원하는 검색 키워드를 넣어서 검색하고 2페이지 이후부터는 같은 endpoint를 사용하지만 파라메터 형식이 완전히 다른 별개의 API를 호출하는 방식으로 되어 있었다. 심지어 2페이지 이후 검색 모드에서는 검색 키워드를 다시 넣는 것도 불허한다. 오직 `rid`만 입력받는다.

이 경우 호출 측에서 검색 키워드, 페이지 번호, rid값을 기억하고 각각의 경우에 대해 API call을 적용할 수도 있지만(실제로 제네레이터를 쓰기 이전 버전에서는 그렇게 했다) 이렇게 제네레이터 함수로 작성할 경우 호출 측에서는 페이지 번호에 신경쓰지 않고 `for await` 루프를 통해 submission_id 값을 제공받을 수 있다.

해당 제네레이터를 사용하는 부분에서는 이렇게 사용한다.

```javascript
submissionList = fetchSubmissionList(token, {
  favs_user_id: token.user_id,
  orderby: "fav_datetime",
});

for await (const submissionId of submissionList) {
  console.log(`#${submissionId}`);
  for await (const fileInSubmission of getSubmission(token, submissionId)) {
    // ... 한 submission 안에는 여러 개의 file이 있을 수 있음(comic book 등)
  }
}
```

## 화살표 함수?

제네레이터 함수는 화살표 함수로 정의하는 것이 불가능하다. 반드시 `function*` 키워드로 함수를 생성해야 한다.
