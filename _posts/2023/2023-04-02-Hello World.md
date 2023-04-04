---
category: 2023
tag:
  ["hello_world", "2023", "jekyll", "github_pages", "minimal-mistakes", "dev"]
author_profile: false
sidebar:
  nav: "how-to-github-pages"
---

# Hello World

안녕하세요! 깃허브 블로그의 첫 글입니다.

```python
if __name__ == '__main__':
    print("Hello world!")
```

![200308-Hoya](/assets/images/2023/2023-04-02-Hello%20World/200308-Hoya.png)

## 이 블로그를 preview 모드로 실행하는 법

```powershell
bundle exec jekyll serve
```

http://localhost:4000 에서 서빙된다.

### tzinfo 관련 오류 해결

https://honsal.blogspot.com/2015/12/tzinfo.html

현재 프로젝트의 gemfile을 열고 아래 내용을 추가한다.

```
# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem 'tzinfo'
gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw]
```

### jekyll-postfiles

상대 경로로 이미지를 서빙할 수 있게 만들어 주는 플러그인이지만, 안타깝게도 github pages와 호환되지 않는 플러그인이다.
Cloudflare Pages 나 그 밖의 정적 웹 호스팅 서비스에서는 정상 작동한다고 한다.

### sass deprecation warning

로컬 환경에서 preview 할 때 자꾸만 sass deprecation warning이 떠서 불편하다. 대부분 나눗셈 연산자가 Dart Sass 2.0.0 이후로는 더 이상 지원되지 않는다는 경고 메시지이다.

```
Deprecation Warning: Using / for division outside of calc() is deprecated and will be removed in Dart Sass 2.0.0.
```

github.io 가 아닌 외부 사이트에 바로 호스팅할 경우에는 jekyll 내부 코드를 원하는대로 수정해서 사용할 수 있지만 이 경우에는 그럴 수 없다.
이 경고를 없애기로 Suppress Warning 한다.

https://www.reddit.com/r/Jekyll/comments/zunif0/help_please_i_keep_getting_deprecation_warnings/

여기에 따르면 Gemfile 의 sass-converter를 다운그레이드하라고 한다.

Gemfile

```
gem "jekyll"
gem "minimal-mistakes-jekyll"
gem "jekyll-sass-converter", "~> 2.0"
```

### TOC

TOC를 추가하는 방법

\_config.yml 파일에 아래 변수 추가

```yaml
# Defaults
defaults:
  # _posts
  - scope:
      path: ""
      type: posts
    values:
      layout: single
      author_profile: true
      read_time: true
      comments: true
      share: true
      related: true
      show_date: true
      toc: true # 이것을 추가
```

### 404 Not Found

\_pages/404.md 파일을 만들고 https://codepen.io/sarazond/pen/jOKyjZ 의 내용을 복사.
frontmatter는 아래 예제를 참고.

페이지 템플릿은 https://freefrontend.com/html-css-404-page-templates/ 여기서 받았습니다.

```md
---
title: "Page Not Found"
excerpt: "Page not found. Your pixels are in another canvas."
sitemap: false
permalink: /404.html
---

<!-- https://codepen.io/sarazond/pen/jOKyjZ -->
```
