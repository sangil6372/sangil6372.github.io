---
layout: post
title: "OPIc 시뮬레이터 — 제출 버튼 Race Condition 수정"
date: 2026-06-09
categories: [개발, 회고]
tags: [OPIc, JavaScript, race-condition, async, 버그수정]
---

## 오늘 한 일

- `_callEvaluateFeedback` 중복 호출 방지를 위한 `rec.evaluating` 플래그 도입
- 최종 제출 버튼의 race condition 수정 — 진행 중인 평가를 기다린 후 저장
- `saveWhenReady` 폴링 함수 추가로 in-flight 요청 완료 대기 처리

## 핵심 작업 해설

### 문제: 중복 호출과 경쟁 상태

GPT-4o Audio 기반 평가 요청은 비동기로 처리된다. 문항 패널을 여닫는 동작이나 제출 버튼 클릭 타이밍에 따라 `_callEvaluateFeedback`이 같은 `rec`에 대해 두 번 이상 호출될 수 있었다. 더 큰 문제는 **제출 버튼**이었다 — 이미 요청이 날아간(`evaluating`) 문항을 `pending` 목록에서 누락시키면서도 그 완료를 기다리지 않고 바로 저장을 시도했다.

### 해결 1: evaluating 플래그

```javascript
function _callEvaluateFeedback(rec) {
    if (!rec || !rec.url) return;
    if (rec.evaluating) return;  // 이미 진행 중 — 재호출 차단
    rec.evaluating = true;

    fetch(rec.url)
        // ...
        .then(function(d) {
            rec.evaluating = false;
            rec.evaluated  = true;
            // UI 업데이트...
        })
        .catch(function() {
            rec.evaluating = false;
            rec.evaluated  = true;  // 실패해도 재시도 방지
        });
}
```

`rec.evaluating` 플래그를 세우면 동일 레코드에 대한 중복 `fetch`가 원천 차단된다. 성공·실패 모두 `evaluating = false`로 해제해야 한다는 점이 중요하다.

### 해결 2: saveWhenReady 폴링

```javascript
var pending = _recs.filter(function(r) {
    return r && r.url && !r.evaluated && !r.evaluating;  // in-flight 제외
});

function saveWhenReady() {
    var inFlight = _recs.filter(function(r) { return r && r.evaluating; });
    if (inFlight.length > 0) {
        $btn.html('🎙️ 분석 완료 대기 중...');
        setTimeout(saveWhenReady, 300);
        return;
    }
    $btn.html('💾 저장 중...');
    _saveAllRecs().then(function() {
        $btn.html('✅ 저장 완료!');
        setTimeout(function() { window.location.href = ltiOpicAppSettings.nextPage; }, 1200);
    });
}
```

`pending`에서 `evaluating` 중인 항목을 제외하되, `evaluateNext` 완료 후 곧바로 저장하는 대신 `saveWhenReady`를 호출한다. 300ms마다 in-flight 요청 수를 확인하고, 모두 완료된 뒤에야 저장을 시작한다.

## 문제 & 해결

**막혔던 부분**: 처음에는 `pending` 필터에서 `evaluating` 항목만 제외하면 충분할 것이라 생각했다. 하지만 제출 버튼을 누른 시점에 이미 in-flight 상태인 평가들이 있을 수 있어, 이들이 완료되기 전에 저장이 진행되면 빈 transcript/feedback이 그대로 저장되는 문제가 남았다.

**해결**: `saveWhenReady` 폴링 함수를 분리해 in-flight가 0이 될 때까지 저장을 지연시키는 방식으로 해결했다. Promise 체이닝이나 별도 카운터보다 단순하고, 300ms 간격은 UX 지연 없이 충분하다.

## 배운 점

- `evaluated` 하나로 "완료 여부"를 표현하면 "진행 중"을 구분할 수 없다 — 상태를 세분화할 것
- 비동기 작업이 여러 경로에서 동일 객체를 수정할 때는 진입 시 플래그, 종료 시 해제를 명시적으로 처리해야 한다
- 폴링은 단순하지만, 짧은 간격(300ms)이면 실용적으로 충분하다

## 다음 작업

- 실제 OPIc 문항 세트 추가 및 난이도 분류
- 결과 페이지 점수 시각화 개선
