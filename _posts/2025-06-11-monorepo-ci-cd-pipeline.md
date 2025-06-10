---
layout: post
title: "FE Core팀의 CI 속도전: 캐시 전략을 활용한 병렬 빌드"
subtitle: Monorepo 환경에서의 빌드 최적화 사례
date: 2025-06-11 00:00:00 +0900
category: fe
background: "/img/2025-06-11-monorepo-ci-cd-pipeline/matrix.png"
author: arnold
comments: true
tags:
  - github workflow
  - turborepo
  - monorepository
---

<br />

# 목차

1. [개요](#1-개요)
2. [기존 파이프라인 구조와 한계](#2-기존-파이프라인-구조와-한계)
   1. [Monorepo 환경의 CI 요구사항](#21-monorepo-환경의-ci-요구사항)
   2. [빌드 시간 및 신뢰성 이슈](#22-빌드-시간-및-신뢰성-이슈)
3. [개선 전략 및 구현](#3-개선-전략-및-구현)
   1. [Runner 사양 개선](#31-runner-사양-개선)
   2. [병렬 빌드(Matrix) 도입](#32-병렬-빌드matrix-도입)
   3. [캐시를 활용한 빌드 최적화](#33-캐시를-활용한-빌드-최적화)
   4. [빌드 검증 단계 분리](#34-빌드-검증-단계-분리)
4. [결과](#4-결과)
5. [후기](#5-후기)

---

<br /><br />

# 1. 개요

안녕하세요 FE Core팀 아놀드입니다.

FE Core팀은 최근 배포 빈도와 변경사항이 많은 monorepo 환경을 관리하며, 효율적이고 안정적인 CI 파이프라인 구축을 목표로 다양한 전략을 도입하였습니다.

본 글에서는 실제로 적용한 빌드 성능 개선, 캐시 활용 방안, 그리고 각 전략의 효과를 정리하였습니다.

# 2. 기존 파이프라인 구조와 한계

## 2.1 Monorepo 환경의 CI 요구사항

현재 turborepo 기반의 mono repository에는 30여 개의 독립적인 상용 프로젝트가 공존하고 있습니다.  
각 프로젝트는 별도의 팀에서 운영하며, 배포 일정과 서비스 특성이 상이합니다. 이로 인해, main 브랜치(main branch)로의 병합이 자주 발생하며, `pnpm-lock.yaml` 파일의 잦은 변경이 수반됩니다.

패키지 의존성 변동을 최소화하기 위해 core 라이브러리 버전을 고정하고 caret(캐럿) 범위 사용을 제한하였으나, 하위 패키지의 caret 사용까지 완전히 통제하는 데에는 한계가 있었습니다. 결과적으로 main 브랜치에 변경이 발생하면, 의존성 변경 여부에 따라 모든 프로젝트에 대한 빌드 및 안정성 검증이 필요합니다.

## 2.2 빌드 시간 및 신뢰성 이슈

CI 파이프라인은 아래와 같은 구조로 운영되고 있었습니다.

- Turborepo 원격 캐시 서버를 별도 운영(Kubernetes 기반)
- `dorny/paths-filter@v3`를 활용하여 불필요한 빌드를 최소화

그럼에도 불구하고 캐시 미적중 시 30개 이상의 Next.js 앱 전체 빌드에 평균 20분 이상이 소요되었습니다. 여러 워크플로우가 동시에 실행되는 경우, 브랜치 병합 대기 시간은 30분 이상으로 증가하였습니다.

또한, 빌드가 길어질 경우 워크플로우가 중단되거나, `Error: The operation was canceled.`와 같은 오류가 빈번하게 발생하였습니다.

아래 이미지는 실제로 파이프라인 실행 시간이 32분을 넘긴 사례입니다. 이처럼 빌드 확인 단계에서 워크플로우가 강제 종료되는 일이 반복되었습니다.

![runtime.png](/img/2025-06-11-monorepo-ci-cd-pipeline/runtime.png)

# 3. 개선 전략 및 구현

빌드 단계를 생략할 수 없으므로, **빌드 시간 단축**이 최우선 과제로 선정되었습니다.  
아래와 같은 방향으로 개선을 추진하였습니다.

## 3.1 Runner 사양 개선

우선, 기존에 사용하던 Ubuntu Runner의 메모리와 코어 수를 상향 조정하였습니다. 이를 통해 파이프라인 전체 실행 시간은 약 20분에서 10분대로 단축되었으며, 중단 오류 없이 안정적으로 빌드가 완료되었습니다.

- 기존 Runner

![originrun.png](/img/2025-06-11-monorepo-ci-cd-pipeline/originrun.png)

- 변경된 Runner

![changerun.png](/img/2025-06-11-monorepo-ci-cd-pipeline/changerun.png)

|      구분      | Runner 사양 변경 전 | Runner 사양 변경 후 |
| :------------: | :-----------------: | :-----------------: |
| 평균 빌드 시간 |       20분대        |       10분대        |

## 3.2 병렬 빌드(Matrix) 도입

Runner 성능 개선 이후에도 단일 프로세스에서 30개 프로젝트 전체를 빌드하는 데 10분 이상이 소요되었습니다. 따라서, [**GitHub Workflow의 Matrix 전략**](https://docs.github.com/ko/enterprise-cloud@latest/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow)을 도입하여 각 프로젝트의 빌드를 병렬로 수행하도록 워크플로우를 개편하였습니다.

Matrix 전략은 다음과 같은 이점을 제공합니다.

- 프로젝트별 빌드를 동시에 진행(병렬화)
- 각 빌드 성공 여부 개별 확인
- 캐시 서버를 활용한 효율적 리소스 분배

**구현 예시**

```yaml
jobs:
  generate-matrix:
    runs-on: ubuntu-22.04-cores
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - name: 저장소 체크아웃
        uses: actions/checkout@v4
      - name: 빌드 대상 패키지 목록 생성
        id: set-matrix
        run: echo "::set-output name=matrix::$(bash .github/scripts/get-packages.sh)"

  build:
    needs: [generate-matrix]
    runs-on: ubuntu-22.04-cores
    strategy:
      matrix: ${{ fromJson(needs.generate-matrix.outputs.matrix) }}
      fail-fast: false
    steps:
      - name: 빌드
        run: pnpm build --filter=${{ matrix.package }}...
```

|      구분      | 전체 빌드 | Matrix 적용 후 |      변화      |
| :------------: | :-------: | :------------: | :------------: |
| 캐시 미적중 시 |  10m27s   |      8m6s      | -2m21s(-22.5%) |
|  캐시 적중 시  |   1m14s   |     6m57s      | +5m43s(463.5%) |

※ **캐시 적중 시에는 병렬화 오버헤드로 인해 빌드 시간이 증가하는 단점이 있었습니다.**

Matrix 전략 적용 후, 각 프로젝트의 빌드 성공 여부를 별도로 확인할 수 있고
아래와 같이 status check가 표시됩니다.

![matrix.png](/img/2025-06-11-monorepo-ci-cd-pipeline/matrix.png)

## 3.3 캐시 최적화

Matrix 병렬 빌드 도입 후, 캐시 적중 시 오히려 시간이 늘어나는 현상을 해결하기 위해 `turborepo`의 [**dry-run**](https://turborepo.com/docs/crafting-your-repository/caching#using-dry-runs) 기능을 활용하였습니다.

![idea.png](/img/2025-06-11-monorepo-ci-cd-pipeline/idea.png)

`turbo run build --dry-run` 명령을 통해 모든 패키지의 캐시 상태를 사전 점검하고, 위 사진의 내용과 같은 구조로 오버헤드를 최소화하였습니다.

- 모든 패키지가 캐시된 경우 빌드 단계를 건너뜀

- 캐시 미적중 패키지만 matrix 대상으로 빌드 실행

프로젝트별 캐시 여부를 확인하여 상황에 따라 빌드 프로세스를 분기하는 구조를 아래와 같이 구현하였습니다.

**구현 예시**

```yaml
- name: turborepo 캐시 확인
  id: check-cache
  run: ./.github/scripts/check-turborepo-cache.sh
- name: 빌드 스킵
  if: ${{ steps.check-cache.outputs.packages-to-build-length == 0 }}
  run: echo "모든 빌드 대상이 캐시되어 있어, 진행하지 않습니다."
```

이를 통해 캐시 적중 시 시간이 늘어나는 현상을 아래와 같이 해결할 수 있게 되었습니다.

|      구분      | 전체 빌드 | Matrix+Dry 적용 후 |      변화      |
| :------------: | :-------: | :----------------: | :------------: |
| 캐시 미적중 시 |  10m27s   |       5m29s        | -4m58s(-47.5%) |
|  캐시 적중 시  |   1m14s   |       1m11s        |    -3s(-4%)    |

## 3.4 빌드 검증 단계 분리

Workflow Matrix를 활용하면, 개별 빌드 결과가 각각의 `status check`로 기록됩니다.
브랜치 보호 정책을 단일 status로 관리하기 위해, 빌드 완료 후 추가 검증 단계를 도입하였습니다.

**구현 예시**

```yaml
verify-build:
needs: build-matrix
runs-on: ubuntu-22.04-4-cores
steps:
  - name: 모든 빌드 성공 확인
run: echo "모든 build-matrix 작업이 성공적으로 완료되었습니다."
```

<br />
# 4. 결과

Runner 사양 개선, 병렬 빌드(Matrix) 도입, 캐시 상태 사전 점검, 빌드 검증 단계 분리 등 여러 전략을 조합함으로써,
전체 CI 파이프라인 빌드 시간을 최대 **84%**까지 단축할 수 있었습니다.

- 빌드 미적중 시 30분대 → 5분대 단축

- 캐시 적중 시 오버헤드 최소화 및 불필요 빌드 방지

- 브랜치 보호 정책을 단일 status로 관리

특히, 프로젝트 수가 증가하더라도 빌드 속도와 안정성의 하락 없이 효율적으로 대응할 수 있는 구조를 마련하였습니다.

# 5. 후기

이번 CI 파이프라인 개선 작업은 단순한 빌드 속도 향상을 넘어,
효율적인 자동화와 체계적인 캐시 전략의 중요성을 다시 한 번 실감하는 계기가 되었습니다.

여러 실험과 시행착오를 통해 실제로 체감할 수 있는 성능 개선 효과를 얻었으며,
이 경험을 바탕으로 앞으로도 더 나은 개발 환경을 만들어 나가고자 합니다.

앞으로 프로젝트 규모가 커지더라도 안정적이고 유연하게 대응할 수 있도록, 지속적으로 자동화와 최적화 방안을 탐구할 계획입니다.
