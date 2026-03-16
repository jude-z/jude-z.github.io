---
title: "Git Rebase vs Merge, 언제 뭘 써야 할까"
categories:
  - Dev Notes
tags:
  - Git
lang: ko
date: 2026-03-17
excerpt: "rebase와 merge의 차이점, 그리고 실무에서 어떤 상황에 어떤 전략을 쓰는 게 맞는지 정리."
---

## Merge

두 브랜치의 히스토리를 합치면서 **merge commit**을 만든다.

```bash
git checkout main
git merge feature
```

- 히스토리가 있는 그대로 보존됨
- merge commit이 쌓여서 로그가 복잡해질 수 있음

## Rebase

현재 브랜치의 커밋들을 대상 브랜치 위로 **재배치**한다.

```bash
git checkout feature
git rebase main
```

- 히스토리가 일직선으로 깔끔해짐
- 이미 push한 커밋을 rebase하면 충돌 위험

## 실무에서의 선택 기준

| 상황 | 추천 |
|------|------|
| feature 브랜치를 main에 합칠 때 | merge (PR merge) |
| 작업 중 main 변경사항을 가져올 때 | rebase |
| 공유 브랜치 (이미 push됨) | merge |
| 로컬에서만 작업 중 | rebase |

## 핵심 원칙

> **이미 공유된 커밋은 rebase하지 않는다.**

로컬 작업은 rebase로 깔끔하게 정리하고, 공유 브랜치에 합칠 때는 merge를 쓰는 게 안전하다.
