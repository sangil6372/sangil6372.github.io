---
layout: post
title: "OPIc 시뮬레이터에 GPT-4o Audio 기반 STT와 Diagnostic Form 도입"
date: 2026-06-09
categories: [개발, 회고]
tags: [OPIc, GPT-4o, Audio, STT, ACTFL, aiohttp, JavaScript]
---

## 오늘 한 일

- OPIc 공식 Diagnostic Comments Form(Advanced/Intermediate Functions 표)을 `result.html`에 구현
- Web Speech API 방식 STT를 `gpt-4o-audio-preview` 기반으로 교체 — `/stt-evaluate` 엔드포인트 신설
- 발음·유창성 평가 항목(ADV5, INT4 행) 완전 복원
- 버그 3건 수정: `re`/`base64` import 누락, JSON 파싱 불안정, 피드백 타이밍 mismatch

---

## 핵심 작업 해설

### 1. STT 방식의 근본적 전환 — Web Speech API → GPT-4o Audio

OPIc Diagnostic Form에는 **발화 속도(rate_of_speech)**, **유창성(fluidity)**, **억양(intonation)**, **강세(stress)** 등 음성 데이터를 기반으로 평가해야 하는 항목이 있다. 문제는 기존 방식이었다.

Web Speech API는 녹음된 오디오를 텍스트 문자열로만 반환한다. 발음이 어떻든, 속도가 어떻든 최종 결과물은 텍스트뿐이다. 즉, 채점 근거가 사라진 채로 음성 평가 항목을 GPT에게 요청하는 셈이었다.

해결책은 `gpt-4o-audio-preview` 모델을 활용하는 `/stt-evaluate` 엔드포인트 신설이었다. 브라우저가 녹음한 webm 바이너리를 base64로 인코딩해 모델에 직접 전달하면, 모델이 음성을 듣고 전사(transcript)와 발음 문제 목록(pronunciation_issues)을 함께 반환한다.

```python
# tts_server.py — /stt-evaluate 핵심 로직
audio_b64 = base64.b64encode(audio_bytes).decode('utf-8')
resp = client.chat.completions.create(
    model='gpt-4o-audio-preview',
    max_tokens=600,
    messages=[{'role': 'user', 'content': [
        {'type': 'input_audio', 'input_audio': {'data': audio_b64, 'format': 'webm'}},
        {'type': 'text', 'text': prompt_text}
    ]}]
)
```

반환 형식은 JSON으로 강제하며, `pronunciation_issues` 배열에는 `stress`, `intonation`, `rate_of_speech`, `fluidity` 등의 키를 담는다. 이 정보가 `/overall-feedback` 호출 시 각 문항의 `음성 분석:` 라인으로 전달되어 Diagnostic Form 채점에 반영된다.

### 2. OPIc Diagnostic Comments Form 재현

ACTFL 공식 채점지 형식을 그대로 HTML 테이블로 구현했다. Advanced Functions 5행(ADV1~ADV5), Intermediate Functions 4행(INT1~INT4)이며 각 행마다 performance level(4단계)과 체크박스 항목들이 있다.

Vue 3의 반응형 바인딩으로 GPT-4o가 반환한 `diagnostic` JSON 객체를 각 셀에 연결했다.

```javascript
// result.html — 체크박스 바인딩 헬퍼
function dHas(section, field, key) {
    const d = overall.value?.diagnostic;
    if (!d || !d[section]) return false;
    const arr = d[section][field];
    return Array.isArray(arr) && arr.includes(key);
}
function perfClass(perf) {
    if (perf.includes('Fully'))      return 'perf-fully';
    if (perf.includes('Minimally'))  return 'perf-minimally';
    if (perf.includes('Developing')) return 'perf-developing';
    if (perf.includes('Does Not'))   return 'perf-not-meet';
    return '';
}
```

체크박스는 `□` / `☑` 유니코드를 CSS `::before`로 렌더링한다. 체크된 항목은 primary 색상(#FF6633)으로 강조된다.

### 3. 피드백 타이밍 mismatch 해결

녹음 종료 직후 피드백 패널을 열면 GPT-4o audio STT가 아직 백그라운드에서 실행 중인 상태다. 기존 코드는 이 타이밍에 Web Speech API 텍스트로 즉시 `_fetchFeedback`을 호출해버려서, GPT-4o 전사 결과가 피드백 내용에 반영되지 않는 버그가 있었다.

`_pendingFeedbackIdx` 변수를 도입해 해결했다. STT가 아직 완료되지 않은 문항에 대해 피드백 패널이 열리면 대기 상태로 전환하고, GPT-4o audio 응답이 도착하는 시점에 최종 transcript로 피드백을 요청한다.

```javascript
// test-question.js — _openFeedback 내 타이밍 제어
if (rec.sttPending) {
    var recIdx = _recs.indexOf(rec);
    _pendingFeedbackIdx = recIdx;
    $('#fbAiMain').html('<div class="fb-loading">🎙️ 음성 분석 중... 잠시 후 피드백이 생성됩니다</div>');
} else {
    _fetchFeedback(rec.qText, rec.text);
}

// STT 완료 콜백에서
if (_pendingFeedbackIdx === idx) {
    _pendingFeedbackIdx = -1;
    _fetchFeedback(rec.qText, rec.text);  // 최종 transcript 사용
}
```

---

## 문제 & 해결

**`re` 모듈 누락으로 런타임 에러**

`/stt-evaluate` 엔드포인트에서 `re.search()`를 사용했는데 파일 상단 import에 `re`가 없었다. 서버를 실행하면 함수 정의 시점에는 오류가 없다가 실제 요청이 들어오는 순간 `NameError`로 500이 반환됐다. `import os, json, re, base64, webbrowser, threading` 한 줄로 해결했지만, 실행 전 정적 분석으로 잡지 못한 점이 아쉬웠다.

**GPT 응답 앞뒤 텍스트로 JSON 파싱 실패**

`gpt-4o-audio-preview`는 가끔 JSON 앞뒤에 설명 텍스트를 붙여 반환한다. 마크다운 펜스 제거(`re.sub`)만으로는 부족했고, `re.search(r'\{.*\}', raw, re.DOTALL)`로 JSON 블록만 추출하는 방식으로 강화했다.

---

## 배운 점

1. **STT는 모달리티를 보존해야 한다.** 텍스트로 변환하는 순간 발음·속도·억양 정보는 영구적으로 소실된다. 평가 기준이 음성 특성을 요구한다면 오디오를 원본 그대로 모델에 전달하는 것이 올바른 설계다.
2. **비동기 레이스 컨디션은 상태 변수로 명시적으로 관리해야 한다.** `sttPending` 플래그와 `_pendingFeedbackIdx`처럼 "지금 무엇을 기다리고 있는가"를 명시하면 콜백 체인이 훨씬 추론하기 쉬워진다.
3. **Python의 `NameError`는 함수 호출 시점에 발생한다.** 정의 시점에 잡히지 않으므로, 서버 코드는 import 라인을 작성한 뒤 한 번은 직접 엔드포인트를 호출해 연기된 오류가 없는지 확인하는 습관이 필요하다.

---

## 다음 작업

- Diagnostic Form의 채점 기준을 시스템 프롬프트에 더 정밀하게 기술하여 GPT-4o 출력의 일관성 향상
- `/stt-evaluate` 호출 비용 최적화 — 짧은 답변은 Web Speech API를 유지하고 일정 길이 이상만 audio 모델로 분기하는 방안 검토
- 실제 OPIc 문항 세트 확충 (T9/T10 추가 완료 후 최종 15문제 검증)
