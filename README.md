# minjune8506.github.io

GitHub Pages + Jekyll로 만든 기술 블로그입니다.

## 로컬에서 실행하기

```bash
bundle install
bundle exec jekyll serve
```

브라우저에서 http://localhost:4000 접속

## 새 글 작성

`_posts/YYYY-MM-DD-제목.md` 형식으로 파일을 추가합니다.

```yaml
---
layout: post
title: "글 제목"
date: YYYY-MM-DD HH:MM:SS +0900
categories: [카테고리]
tags: [태그1, 태그2]
---

본문 내용
```

## 배포

`main` 브랜치에 push하면 GitHub Actions(`.github/workflows/pages.yml`)가 자동으로 빌드/배포합니다.

최초 1회, GitHub 저장소의 **Settings → Pages → Build and deployment → Source**를 **GitHub Actions**로 설정해야 합니다.
