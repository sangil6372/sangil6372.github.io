---
layout: post
title: "WorkManager가 몰래 재시작한 작업이 유령 노트를 세 개나 만든 이야기"
date: 2026-08-10
categories: [개발, 트러블슈팅]
tags: [Android, WorkManager, Kotlin, race-condition, idempotency, mutex, 버그수정]
---

## 오늘 한 일

- 녹음 하나를 AI로 정리했는데 완료되지 않은 노트가 3개나 생기고, 4번째 시도가 도는 중에 멈춰 있는 버그를 재현·분석
- 실기기 logcat으로 같은 `WorkManager` 작업 ID가 짧은 간격으로 취소·재시작되는 것을 확인
- 원본 파일(`source`) 기준으로 "정리 중" 노트를 찾아 재사용하는 idempotency 로직 추가
- 같은 원본 파일에 대한 동시 처리를 막는 파일명 단위 락(lock) 추가

## 핵심 작업 해설

### 문제: 같은 녹음인데 노트가 세 개

음성 녹음을 STT로 받아쓰고 AI로 요약해 마크다운 노트로 저장하는 안드로이드 앱을 만들고 있다. 녹음 하나를 정리했더니 기기에 이런 노트 세 개가 남아 있었다:

```
2026-08-10_1631_이런 식으로 이퀄이 짜져 있는 걸 하고 있고.md
2026-08-10_1631_이런 식으로 짜여 있는 걸 하고 있고, PBI는 그냥….md
2026-08-10_1631_이런 식으로 쫙 짜여 있는 걸 서비스거든요.md
```

frontmatter를 열어보니 제목만 다를 뿐 `source`(원본 파일명)·`created`(녹음 시각)·`duration`(길이)이 완전히 같았다. 그리고 셋 다 `type: "정리 중"` — 즉 전사(STT)까지만 끝나고 요약이 안 붙은 채로 멈춰 있었다. **하나의 녹음이 세 번 처리를 시도했고, 세 번 다 중간에 멈췄다는 뜻이다.**

이미 이 앱은 파일 하나당 작업 하나(`WorkManager`의 `enqueueUniqueWork` + `ExistingWorkPolicy.KEEP`)로 묶어서 같은 파일이 중복 큐잉되는 걸 막아 뒀었다. 그런데도 이런 일이 벌어졌다는 건, 문제가 "중복 큐잉"이 아니라 다른 곳에 있다는 뜻이었다.

### 원인: 재시도가 매번 새 파일을 만든다

logcat을 걸러보니 결정적인 단서가 나왔다. 같은 작업 ID(`d31709fe-...`)가 몇 초 간격으로 반복해서 취소·재시작되고 있었다.

```
16:39:42.658  Starting work for UploadWorker
16:39:42.979  전사 시작: memo-20260810-162144.m4a
16:39:44.338  Work [ id=d31709fe... ] was cancelled
16:39:44.416  Starting work for UploadWorker        ← 같은 프로세스, 곧바로 재시작
16:39:49.453  Work [ id=d31709fe... ] was cancelled
16:39:54.563  전사 완료 노트 먼저 저장: ...서비스거든요.md
16:39:55.312  전사 완료 노트 먼저 저장: ...이퀄이 짜져 있는 걸...md
```

시스템이 백그라운드 작업을 중간에 끊고(다른 앱으로 전환하는 등의 이유로 워커가 강제 종료·재스케줄되는 건 `CoroutineWorker`가 흔히 겪는 문제다) 다시 시작시키는데, **끊긴 것으로 표시된 이전 시도의 코루틴이 실제로는 멈추지 않고 백그라운드에서 계속 돌고 있었다.** 그러다 보니 취소된 시도와 새로 시작한 시도가 동시에 전사를 끝내고, 각자 "정리 중" 노트를 만들어 버린 것이다.

기존 코드는 전사가 끝나면 먼저 "정리 중" 상태의 임시 노트를 만들고, 요약이 끝나면 **같은 파일을 덮어써서** 업데이트하는 구조였다.

```kotlin
val interim: File? = if (summarize) {
    LocalNotes.save(
        context = context,
        title = titleFromTranscript(transcript, audio.nameWithoutExtension),
        body = "",
        noteType = "정리 중",
        // ...
    )
} else null
```

문제는 `overwrite`를 지정하지 않아 **항상 새 파일**을 만든다는 점이었다. 게다가 파일명이 `titleFromTranscript()` — 즉 전사문 첫 문장에서 뽑아낸다. STT는 결정적이지 않아서 같은 오디오를 다시 돌려도 받아쓰기 결과가 살짝 달라질 수 있는데, 그러면 "이전 시도가 만든 노트를 파일명으로 찾는" 것 자체가 불가능하다. 실제로 세 노트의 제목이 전부 미묘하게 달랐던 이유가 이거였다.

### 해결 1: source로 이전 시도의 노트를 찾는다

제목은 매번 바뀌어도 **원본 오디오 파일명(`source`)은 재시도해도 절대 안 바뀐다.** 이걸 idempotency 키로 썼다.

```kotlin
/**
 * 같은 원본(source)의 "정리 중" 노트가 이미 있으면 그 파일을 준다.
 * type 이 "정리 중" 인 것만 찾는다 — 이미 끝난 노트를 재시도 경로가 건드리면 안 된다.
 */
fun findStuckInterim(context: Context, sourceName: String): File? =
    dir(context).listFiles { f -> f.extension == "md" }
        ?.filter { fieldOf(it, "source") == sourceName && fieldOf(it, "type") == "정리 중" }
        ?.maxByOrNull { it.lastModified() }
```

그리고 임시 노트를 저장할 때 이 함수로 먼저 찾아본 뒤, 있으면 덮어쓰게 바꿨다.

```kotlin
overwrite = LocalNotes.findStuckInterim(context, audio.name),
```

이제 재시도가 몇 번이 나든 "정리 중" 노트는 하나만 존재한다.

### 해결 2: 같은 파일은 한 번에 하나만

그런데 이것만으로는 부족했다. logcat에서 본 것처럼 **두 시도가 진짜로 동시에** 전사를 끝낼 수 있다. 그 타이밍에 걸리면 둘 다 `findStuckInterim()`을 호출했을 때 아직 아무도 저장하지 않은 상태라 둘 다 "없음"으로 보고, 각자 새 파일을 만들어버린다. 확인(check)과 생성(create) 사이에 원자성이 없는 전형적인 TOCTOU(Time-Of-Check-To-Time-Of-Use) 레이스였다.

파일명 하나당 락 하나로 이 구간 자체를 직렬화했다.

```kotlin
private val locks = ConcurrentHashMap<String, Any>()

fun process(context: Context, audio: File, /* ... */): Result {
    val lock = locks.computeIfAbsent(audio.name) { Any() }
    return synchronized(lock) {
        processLocked(context, audio, /* ... */)
    }
}
```

`process()`는 원래 suspend 함수가 아니라 백그라운드 스레드에서 블로킹으로 도는 함수였기 때문에, `kotlinx.coroutines.sync.Mutex` 대신 평범한 `synchronized`로 충분했다. 같은 파일에 대한 두 번째 호출은 첫 번째가 끝날 때까지 기다렸다가 시작하고, 이때는 이미 `findStuckInterim()`이 첫 번째 시도의 노트를 찾아줄 수 있다.

## 문제 & 해결

**막혔던 부분**: 처음엔 "재시도 로직에서 파일을 새로 안 만들면 되겠지"라고 단순하게 생각했다. 그런데 실제 logcat을 뜯어보니 순차적인 재시도 말고 **동시에 두 개가 살아있는** 경우가 있었다 — WorkManager가 "취소했다"고 로그를 남긴 작업의 코루틴이 실제로는 안 죽고 계속 돌고 있었던 것이다. idempotency 키만으로는 이 레이스를 못 막는다는 걸 로그를 직접 보고서야 알았다.

**해결**: idempotency(같은 소스면 같은 파일을 찾아 재사용)와 상호배제(같은 소스는 한 번에 하나씩만 처리)를 별개의 문제로 나눠서 각각 풀었다. 하나만으로는 안 되고 둘 다 있어야 완전히 막힌다.

## 배운 점

- WorkManager의 "작업이 취소됐다"는 로그를 그 작업의 실제 코루틴이 멈췄다는 뜻으로 오해하면 안 된다. 취소 신호와 실제 실행 종료 사이에는 간극이 있을 수 있다.
- 재시도 가능한 작업을 설계할 때는 "재시도해도 안전한가(idempotent한가)"와 "동시에 두 개가 돌면 안전한가(thread-safe한가)"를 따로 검증해야 한다. 하나를 고쳤다고 다른 하나가 저절로 해결되지 않는다.
- 파일명처럼 매번 재계산되는 값을 키로 쓰면 재시도 탐지 자체가 실패할 수 있다. 재시도해도 절대 바뀌지 않는 값(여기선 원본 파일명)을 키로 골라야 한다.
- 로그 하나 없이 추측만으로 고쳤다면 idempotency 수정까지만 하고 끝냈을 것이다. 실제 기기 로그를 시간순으로 재구성해보고 나서야 레이스 컨디션까지 있다는 걸 알았다.

## 다음 작업

- 애초에 왜 백그라운드에서 작업이 끊기고 재시작되는지(Android의 백그라운드 실행 제약)는 아직 근본적으로 해결하지 못했다. `setExpedited()` 기반의 우선순위 승격을 시도해볼 계획이다.
