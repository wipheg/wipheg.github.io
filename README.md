# won.dev · Chirpy 개발 블로그

**실제 Jekyll Chirpy 테마**를 사용하는 개인 개발 블로그입니다. 공식 [Chirpy Starter](https://github.com/cotes2020/chirpy-starter)를 바탕으로 설정했습니다.

- 블로그 이름: **won.dev**
- 작성자 / GitHub: **wipheg**
- 공개 예정 주소: **https://wipheg.github.io**
- 한국어 메뉴, 서울 시간대, 밝은 / 어두운 모드
- 카테고리, 태그, 글 보관함, 검색, 글 목차, 코드 강조, RSS

세 개의 글과 소개는 시작을 위한 예시입니다. 글 제목에 `[예시]`라고 표시했습니다. 원하는 내용으로 바꾸거나 삭제하세요.

## GitHub Pages에 올리기

1. GitHub에 **wipheg.github.io**라는 이름의 **Public** 저장소를 만듭니다. 같은 이름의 저장소가 이미 있다면 기존 파일을 먼저 확인하고 백업하세요.
2. 이 폴더 안의 파일들을 저장소 최상위에 올립니다. 바깥 폴더나 ZIP 자체를 올리지 않습니다. **`.github/workflows/pages-deploy.yml`도 반드시 포함**하세요.
3. 저장소 **Settings → Pages → Build and deployment → Source**에서 **GitHub Actions**를 선택합니다.
4. **Actions → Build and Deploy → Run workflow**를 누릅니다. 이후 `main` 브랜치에 글을 추가하면 자동으로 다시 배포합니다.
5. 완료 후 **https://wipheg.github.io**에 접속합니다. 이 파일을 받는 것만으로 인터넷에 공개되는 것은 아닙니다.

GitHub Desktop을 사용하면 숨김 폴더 `.github`도 함께 올릴 수 있습니다. 브라우저 업로드를 사용한다면 `.github/workflows/pages-deploy.yml`이 실제로 올라갔는지 확인하세요.

터미널을 사용한다면 이 폴더에서 다음 명령으로 처음 올릴 수 있습니다. 대상은 새로 만든 빈 저장소입니다.

```sh
git init -b main
git add .
git commit -m "Set up won.dev with Chirpy"
git remote add origin https://github.com/wipheg/wipheg.github.io.git
git push -u origin main
```

이전 Node 기반 블로그를 이미 올린 경우, 이전 `build.mjs`, `package.json`, `package-lock.json`, `site.json`, `posts/`, 기존 `assets/`, `.github/workflows/pages.yml`은 이 테마와 함께 사용하지 않습니다. 기존 글과 이미지를 백업한 뒤 이 Chirpy 프로젝트로 교체하세요. 두 배포 작업을 동시에 실행하지 마세요.

## 글 쓰기

글은 `_posts/날짜-영문제목.md` 파일로 만듭니다. 예: `_posts/2026-11-10-my-first-post.md`.

````markdown
---
title: 오늘 배운 것
date: 2026-11-10 00:00:00 +0900
categories: [개발, JavaScript]
tags: [javascript, 기록]
author: wipheg
toc: true
comments: false
---

## 오늘 해결한 문제

본문을 여기에 적습니다.

```javascript
console.log('Hello, blog!');
```
````

미래 시각의 글은 그 시각이 되기 전까지 목록에 표시되지 않습니다. 이미지는 `assets/img/`에 넣고 `![설명](/assets/img/example.png)`으로 삽입합니다.

## 자주 바꾸는 곳

| 파일 | 내용 |
| --- | --- |
| `_config.yml` | 이름, 짧은 소개, 주소, 언어, 시간대, 프로필 이미지 |
| `_tabs/about.md` | 소개 페이지 |
| `_posts/` | 글 원본 |
| `_data/authors.yml` | 작성자 정보 |
| `_data/contact.yml` | 왼쪽 아래 GitHub·RSS 링크 |
| `assets/img/avatar.svg` | 기본 W 프로필 이미지 |
| `.github/workflows/pages-deploy.yml` | 자동 생성·검사·배포 |

댓글과 방문 통계는 연결하지 않았습니다. 댓글을 사용하려면 본인의 giscus 등 서비스 설정을 `_config.yml`에 입력하세요. 개인 이메일은 임의로 넣지 않았습니다.

## 컴퓨터에서 실행하기

Ruby 3.4와 Bundler가 있는 환경에서 실행합니다. Windows는 공식 Chirpy 안내의 Dev Container 방식을 사용할 수 있습니다.

```sh
bundle install
bundle exec jekyll serve --no-watch
```

브라우저에서 **http://127.0.0.1:4000**을 엽니다. `--no-watch`는 Windows에서 선택적인 감시 도구 없이 실행하기 위한 설정입니다. 변경 후에는 다시 실행하세요.

운영용 파일만 만들려면 `bundle exec jekyll build`를 실행합니다. `_site/`에 결과가 만들어집니다. GitHub에서는 이 과정을 자동으로 수행하므로 컴퓨터에 Ruby를 설치하지 않고도 웹에서 글을 고쳐서 게시할 수 있습니다.

## 출처와 라이선스

- 테마: [cotes2020/jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)
- 시작 템플릿: [cotes2020/chirpy-starter](https://github.com/cotes2020/chirpy-starter)
- 사용법: [Chirpy Getting Started](https://chirpy.cotes.page/posts/getting-started/)

공식 템플릿의 `LICENSE`를 유지했습니다. 기존 블로그 링크의 글이나 프로필 이미지를 복사하지 않았습니다.

## 바로 열어보는 미리보기

함께 제공된 `preview/index.html`을 열면 실제 Chirpy로 생성한 화면을 볼 수 있습니다. 파일로 열 수 있도록 링크를 바꾸고 검색 데이터를 포함한 미리보기 전용 사본입니다. 글 목록, 본문, 태그, 검색, 색상 모드를 확인할 수 있습니다. 폰트와 일부 외부 라이브러리는 인터넷 연결이 필요합니다.

게시할 원본은 `_config.yml`, `_posts/`, `_tabs/` 등입니다. `preview/`는 자동 배포 대상과 Git 추적에서 제외했습니다. 원본 글을 수정해도 이 미리보기 사본은 자동 갱신되지 않습니다. 이후 변경은 GitHub Pages에서 확인하거나 Jekyll 서버를 실행해 확인하세요.
