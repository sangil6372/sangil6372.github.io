---
layout: post
title: "Jetpack Compose에서 딱 한 프레임만 잘못된 탭이 보였던 이유"
date: 2026-08-10
categories: [개발, 트러블슈팅]
tags: [Android, Jetpack Compose, LaunchedEffect, recomposition, UI버그, Kotlin]
---

## 오늘 한 일

- 노트 화면을 열면 잠깐 엉뚱한 탭이 보였다가 곧바로 원래 탭으로 바뀌는 깜빡임 현상 확인
- `LaunchedEffect`로 상태 기본값을 정하는 방식이 근본 원인이라는 것을 파악
- effect 대신 매 recomposition마다 즉시 계산하는 방식으로 교체

## 핵심 작업 해설

### 문제: 화면이 열리자마자 탭이 튄다

노트 화면은 핵심 정리 / 상세 정리 / 원문 스크립트, 이렇게 세 개의 탭을 가질 수 있다. 노트마다 어떤 탭이 존재하는지가 달라서(예: 원문이 없는 노트는 스크립트 탭이 없다), 화면을 열면 그 노트에 맞는 기본 탭을 골라 보여줘야 한다.

처음 구현은 이랬다.

```kotlin
var tab by remember { mutableStateOf(0) }

LaunchedEffect(core, hasSummary, transcript) {
    tab = when {
        core != null -> 0
        hasSummary -> 1
        transcript != null -> 2
        else -> 1
    }
}
```

실기기에서 열어보니, 노트를 열 때마다 아주 짧게 스크립트 탭이 보였다가 곧바로 원래 있어야 할 탭(예: 정리 탭)으로 튀는 게 보였다. 사용자 입장에선 "탭 밑줄이 순간적으로 이상한 곳에 있다가 옮겨간다"는 어색한 깜빡임이었다.

### 원인: effect는 recomposition이 끝난 다음에 실행된다

`LaunchedEffect`는 Compose의 이펙트(effect)다. 이펙트는 **그 recomposition이 끝난 뒤**에 실행되도록 설계돼 있다. 즉 노트 데이터(`core`, `hasSummary`, `transcript`)가 막 로드된 바로 그 첫 프레임에서는, `tab`은 아직 이펙트가 실행되기 전이라 **직전 값(대개 초기값인 0)** 을 그대로 갖고 있다.

화면을 그리는 조건문은 대략 이런 구조였다.

```kotlin
if (tab == 0 && core != null) { CoreTab() }
else if (tab == 1 && hasSummary) { SummaryTab() }
else { ScriptTab() }
```

`core`가 없는(대부분의) 노트에서는 `tab`의 초기값 0이 첫 번째 조건(`tab==0 && core!=null`)에도, 두 번째 조건에도 걸리지 않는다. 그래서 **맨 마지막 else, 즉 스크립트 탭으로 떨어진 채 첫 프레임이 그려진다.** 그다음 프레임에서야 이펙트가 실행되어 `tab = 1`로 바뀌고, 화면이 정리 탭으로 다시 튄다 — 정확히 실기기에서 본 그 현상이었다.

### 해결: 대입하지 말고 매번 계산한다

수정 방향은 간단했다. 기본값을 이펙트로 "나중에 대입"하는 대신, **매 recomposition마다 즉시 계산**해서 첫 프레임부터 맞는 값이 나오게 바꿨다.

```kotlin
// 사용자가 직접 고른 탭(없으면 null)
var explicitTab by rememberSaveable(noteId) { mutableStateOf<Int?>(null) }

val defaultTab = when {
    core != null -> 0
    hasSummary -> 1
    transcript != null -> 2
    else -> 1
}

val tab = explicitTab?.takeIf(::isValidTab) ?: defaultTab
```

`tab`이 콘텐츠가 로드되는 바로 그 프레임부터 옳은 값으로 계산되기 때문에, 잘못된 탭이 그려질 프레임 자체가 존재하지 않는다. 노트를 다시 정리해서 형식이 바뀌어(예: 핵심 정리가 있던 노트가 강의노트로 바뀌어 핵심 탭이 사라짐) 사용자가 골라둔 탭이 더 이상 유효하지 않게 되어도, `isValidTab`이 걸러서 자동으로 기본값으로 돌아간다.

## 문제 & 해결

**막혔던 부분**: 처음엔 "이펙트에 넣은 조건(`core`, `hasSummary`, `transcript`)이 잘못됐나"를 의심하며 조건문을 계속 손봤다. 하지만 조건 자체는 맞았다 — 문제는 조건이 아니라 **그 조건이 언제 평가되는지**였다. 로직을 아무리 고쳐도 "값이 확정되기 전의 프레임"이 존재하는 한 깜빡임은 없어지지 않았다.

**해결**: "이 값을 나중에 맞다고 확정한다" 방식에서 "이 값을 지금 갖고 있는 정보로 즉시 계산한다" 방식으로 접근 자체를 바꿨다. Compose에서 첫 렌더에 보여줄 값이 나중에야 확정되는 구조는, 로직이 아무리 옳아도 반드시 한 프레임의 오류를 만든다는 걸 이때 알았다.

## 배운 점

- Compose의 `LaunchedEffect`, `SideEffect` 같은 이펙트는 recomposition이 끝난 **다음**에 실행된다. "이 프레임에서 바로 반영되는 값"이 필요하다면 이펙트로 상태를 대입하는 방식은 애초에 맞지 않는다.
- 첫 렌더에 보여줄 기본값이 여러 조건에 따라 달라진다면, `remember`에 초기값을 넣고 이펙트로 나중에 고치기보다는 매번 순수하게(부작용 없이) 계산하는 편이 안전하다. 계산 비용이 크지 않다면 "매번 다시 계산"이 "한 번 계산해서 나중에 갱신"보다 버그가 적다.
- 화면이 아주 짧게 깜빡이는 버그는 스크린샷 한 장으로는 안 잡힌다. 실기기에서 직접 눈으로 보거나 화면 녹화를 프레임 단위로 뜯어봐야 재현·확인이 된다.
