---
layout: post
title: "OPIc 시뮬레이터 — 조용히 데이터를 잃던 버그 세 개"
date: 2026-08-26
categories: [개발, 회고]
tags: [OPIc, 버그수정, localStorage, fetch, HTML파싱, Whisper]
---

## 오늘 한 일

- 40분짜리 응시가 통째로 사라지던 문제 수정
- `fetch` 가 500에서 reject 하지 않아 실패한 평가가 재시도되지 않던 문제 수정
- 짝 없는 `</style>` 로 CSS가 화면 좌상단에 노출되던 문제 수정
- Whisper 전사 잔여물 4건 정리 + 감사 스크립트 추가

## 핵심 작업 해설

### 시험을 다 치고 나서 결과가 없다

`tts_server` 를 끄고 화면을 하나씩 돌려 보다 발견했다. 최종 제출 코드는 이랬다.

```javascript
return Promise.all(fetches).then(function() {
    return fetch(OPIcConfig.API.saveRecordings, { method: 'POST', body: form });
}).then(function(r) { return r.json(); }).then(function(d) {
    if (!d.ok) throw new Error(d.error || '저장 실패');
    localStorage.setItem('opic_session_result', JSON.stringify(sessionData));  // ← 여기
});
```

`localStorage` 저장이 **서버 응답 성공 뒤에만** 실행된다. 서버가 죽어 있으면 fetch가
reject되고, 세션은 저장되지 않고, 결과 화면은 빈 화면이 된다. 40분짜리 응시와
**이미 과금된 15문항 피드백**이 함께 사라진다.

녹음 아카이브는 부가 기능이지 결과의 전제가 아니다. 순서를 뒤집었다.

```javascript
persist('');   // 서버가 죽어 있어도 결과는 볼 수 있게

return Promise.all(fetches).then(/* ... */).then(function(d) {
    if (!d.ok) throw new Error(d.error || '저장 실패');
    persist(d.session);   // 서버가 준 세션 ID로 갱신
});
```

실패 시 동선도 고쳤다. 예전에는 alert 후 버튼을 되살려 사용자를 붙잡아 뒀는데,
지금은 "녹음은 보관 못 했지만 채점 결과는 있습니다"라고 알리고 결과 화면으로 보낸다.

### fetch는 500에서 throw하지 않는다

병렬 평가를 테스트하려고 스텁을 500으로 떨어뜨렸는데, 재평가가 0건이었다.

```javascript
.then(function(d) {
    rec.evaluating = false;
    rec.evaluated  = true;   // ← ok=false 인데도 '평가 완료'
    if (d && d.ok) { /* 결과 반영 */ }
});
```

`fetch` 는 HTTP 500에도 정상 resolve한다. `catch` 는 아예 안 타고, `evaluated` 가
세워지면 그 문항은 **영영 재시도되지 않고 피드백 없이 저장**된다.

```javascript
rec.evaluated = !!(d && d.ok);   // 성공했을 때만
```

### 화면 좌상단에 CSS가 보인다

"step 3부터 `@keyframes element-queries { 0% { visibility: inherit; } }` 이게 뜬다"는
제보를 받았다. survey 화면에서는 재현되지 않았다. OPIc 흐름의 3번째가 **Pre-Test Setup**
이었다.

```bash
$ grep -c '<style' pages/setup.html      # 5
$ grep -c '</style>' pages/setup.html    # 6  ← 하나 더
```

navbar `<link>` 를 끼워 넣다가 그 줄이 통째로 복제돼 있었다.

```html
@keyframes element-queries {...}</style><link rel="stylesheet" href="/assets/css/navbar.css">
@keyframes element-queries {...}</style></head>   ← 이 줄이 잉여
```

짝 없는 `</style>` 가 스타일 블록을 일찍 닫으니 남은 CSS가 `<head>` 의 **텍스트 노드**가
되고, HTML 파서가 그걸 `<body>` 맨 앞으로 옮긴다. 그래서 좌상단에 보였다.

**JS 에러도 404도 아니라서 기존 검사를 전부 통과했다.** `test_pages.py` 에 본문 CSS
누출 검사를 추가했다.

## 문제 & 해결

### 전사에 다른 문항이 섞여 있었다

`공원` T4 문제를 읽던 중 발견됐다.

```
4. Explain to me the story of an experience you had while going to a park.
   ... make that trip so unforgettable. 3. Explain to me the story of an
   experience you had while going to a park.   ← 잔여물
```

원본 음원 전사는 Whisper로 한 번에 돌렸는데, 음원 끝의 반복 낭독이 섞여 들어갔다.
눈으로 680건을 훑을 수는 없으니 감사 스크립트를 썼다.

- 본문 중간/끝에 `N.` 으로 시작하는 다른 문항 조각
- 같은 문장이 한 전사 안에 두 번
- 유형 번호와 앞 번호 불일치 (T4인데 "3." 로 시작)
- 같은 주제 다른 문항의 원문이 통째로 포함

12건이 걸렸고 실제 오염은 4건(공원 T4, 음악 T1, 식당 T3, 약속 T1)이었다.
`custom/` 음원은 전사 텍스트로 TTS 생성한 것이라 음원에도 조각이 들어 있어서,
텍스트를 고친 뒤 음원을 다시 만들었다.

## 배운 점

- **성공 경로에 데이터 저장을 묶지 말 것.** 부가 기능(아카이브)이 실패했다고
  본질(응시 결과)을 잃으면 안 된다.
- `fetch` 는 네트워크 실패에만 reject한다. `ok` 를 직접 보지 않으면 500을 성공으로
  센다. 그 결과가 "조용히 넘어감"이라 더 위험하다.
- JS 에러도 404도 아닌 버그가 있다. 검사 항목이 "무엇이 잘못될 수 있나"를 못 따라가면
  테스트가 전부 초록이어도 화면은 깨져 있다.
- 데이터 오염은 **자동으로 훑는 규칙**을 만들어야 잡힌다. 4건을 눈으로 찾은 게 아니라,
  한 건을 제보받고 규칙을 만들어 나머지 3건을 찾았다.

## 다음 작업

- localStorage 내구성 — 지금은 브라우저 기록을 지우면 학습 자료가 사라진다
