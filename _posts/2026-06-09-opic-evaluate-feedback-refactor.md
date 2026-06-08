---
layout: post
title: "OPIc 시뮬레이터 평가 아키텍처 리팩토링 — /evaluate-feedback 통합 엔드포인트"
date: 2026-06-09
categories: [개발, 회고]
tags: [OPIc, GPT-4o, Audio, 리팩토링, aiohttp, JavaScript, 비용최적화]
---

## 오늘 한 일

- 기존 `/stt-evaluate` + `/feedback` 2-call 구조를 `/evaluate-feedback` 단일 호출로 통합
- 피드백 버튼 클릭 시에만 평가 요청, Next 클릭은 blob 보관으로 변경
- 결과 제출 전 미평가 문항 순차 처리 로직 추가
- 캐시 재사용 — 같은 문항 피드백 패널을 다시 열면 추가 API 호출 없음

---

## 핵심 작업 해설

### 1. 아키텍처 문제 인식

기존 구조에는 근본적인 이중성이 있었다.

```
녹음 종료 → /stt-evaluate (audio → transcript + pronunciation_issues)
피드백 열기 → /feedback (transcript 텍스트 → OPIc 피드백)
```

`/stt-evaluate`는 GPT-4o audio-preview를 사용해 오디오를 직접 듣고 발음을 평가한다. 그런데 그 결과로 나온 transcript(텍스트)를 다시 `/feedback`에 넣어 평가를 요청한다. GPT-4o audio가 이미 음성을 들었는데, 그 내용을 텍스트로 변환해서 다른 모델에게 다시 설명하는 꼴이다.

더 큰 문제는 타이밍이었다. `/stt-evaluate`는 녹음 종료 즉시 백그라운드에서 실행된다. 유저가 그 문항의 피드백을 아예 안 열고 Next만 누르면 이미 나간 요청은 낭비다. 13문항 중 절반만 피드백을 확인해도 나머지 6~7개 문항의 STT 비용이 그대로 나간다.

### 2. /evaluate-feedback 통합 엔드포인트

하나의 gpt-4o-audio-preview 호출로 세 가지를 한번에 처리한다.

```python
# tts_server.py
async def evaluate_feedback(req):
    # multipart: audio(webm) + questionText
    audio_b64 = base64.b64encode(audio_bytes).decode('utf-8')

    prompt_text = (
        "Return ONLY valid JSON:\n"
        '{"transcript":"...","pronunciation_issues":[...],"feedback":"...Korean OPIc feedback..."}\n'
        # pronunciation_issues 항목 목록 + feedback 마크다운 형식 지시
    )

    resp = client.chat.completions.create(
        model='gpt-4o-audio-preview',
        max_tokens=2500,
        messages=[{'role': 'user', 'content': [
            {'type': 'input_audio', 'input_audio': {'data': audio_b64, 'format': 'webm'}},
            {'type': 'text', 'text': prompt_text}
        ]}]
    )
    result = json.loads(re.search(r'\{.*\}', raw, re.DOTALL).group(0))
    return web.json_response({
        'ok': True,
        'transcript': result.get('transcript', ''),
        'pronunciation_issues': result.get('pronunciation_issues', []),
        'feedback': result.get('feedback', '')
    })
```

텍스트 변환 단계가 사라졌다. 오디오 → 평가 결과가 직선으로 연결된다.

### 3. 지연 평가(Lazy Evaluation) 전략

유저 액션에 따라 평가 시점을 명확히 분리했다.

```javascript
// test-question.js

// 녹음 종료 — 요청 없음, 상태만 저장
_recs[capturedIdx] = {
    url:       url,
    text:      savedText,   // Web Speech API 임시 텍스트
    evaluated: false,       // 아직 /evaluate-feedback 미호출
    pronunciation_issues: [],
    feedback:  ''
};

// 피드백 패널 열기
function _openFeedback(rec) {
    if (rec.evaluated) {
        // 캐시 재사용 — 추가 API 호출 없음
        _renderFeedback(rec.feedback);
    } else {
        _callEvaluateFeedback(rec);  // 처음 열 때만 호출
    }
}
```

결과 제출 시에는 미평가 문항들을 순차 처리하고 진행 상황을 표시한다.

```javascript
// 결과 제출 버튼
var pending = _recs.filter(function(r) { return r && r.url && !r.evaluated; });
// "🎙️ 음성 분석 중... (2/5)" 식으로 카운트 표시하며 순차 처리
```

---

## 문제 & 해결

**JSON 내 긴 피드백 텍스트의 이스케이프 문제**

`feedback` 필드에 마크다운 텍스트(줄바꿈, 특수문자)가 들어가면 GPT가 JSON을 잘못 닫는 경우가 있다. `re.search(r'\{.*\}', raw, re.DOTALL)` 패턴으로 JSON 블록을 추출하면 어느 정도 방어가 되지만, 모델이 줄바꿈을 `\\n`으로 제대로 이스케이프하지 않으면 `json.loads`가 실패한다. max_tokens를 2500으로 충분히 확보하고 프롬프트에서 피드백 형식을 간결하게 지시해 토큰 낭비를 줄였다.

---

## 배운 점

1. **멀티모달 모델은 중간 변환 단계를 없앨 수 있다.** 오디오 → 텍스트 → 재평가의 연쇄는 정보 손실이 일어나는 구조다. 모달리티를 원본 그대로 모델에 전달하는 것이 정확도와 효율 모두에서 유리하다.
2. **비용 최적화의 핵심은 "필요할 때만 호출"이다.** 백그라운드 선제 호출은 UX를 빠르게 보이게 만들지만, 유저가 그 결과를 쓰지 않으면 낭비다. 지연 평가 + 캐시 패턴이 더 합리적이다.
3. **상태 플래그는 단순하게.** `sttPending`, `_pendingFeedbackIdx` 같은 비동기 타이밍 관리 변수들이 필요했던 이유는 두 개의 비동기 흐름이 서로 의존했기 때문이다. 단일 엔드포인트로 통합하니 상태 관리 코드 100줄이 사라졌다.

---

## 다음 작업

- `/evaluate-feedback` 응답에서 JSON 파싱 실패율 모니터링 (피드백 텍스트가 길어질수록 이스케이프 오류 가능성 존재)
- 결과 제출 시 미평가 문항이 많을 경우 병렬 처리 검토 (현재 순차 처리)
- Diagnostic Form 채점 기준 프롬프트 정밀화
