---
layout: post
title: "KTX 취소표 매크로 — 웹 개발자의 첫 Chrome 확장 프로그램 개발 회고"
date: 2026-06-13
categories: [프로젝트, 회고]
tags: [chrome-extension, manifest-v3, web-worker, web-locks-api, mutation-observer, javascript, korail]
---

## 왜 만들었나

KTX 명절 표는 예매 오픈 직후 수 분 안에 매진된다. 취소표를 노리는 것도 결국 F5를 누르는 속도 싸움인데, 사람이 이걸 직접 하는 건 의미 없다고 생각했다. 웹 개발 경험은 있으니 브라우저에서 직접 동작하는 자동화 도구를 만들어보자고 생각한 게 시작이었다.

Chrome 확장 프로그램은 이번이 처음이었다. 브라우저 API를 코드로 제어한다는 개념 자체는 익숙하지만, 확장 프로그램 특유의 구조와 제약이 있었다.

---

## Chrome 확장 프로그램이 실제로 어떻게 동작하는가

확장 프로그램을 처음 만들 때 가장 먼저 든 의문이 이것이었다. `content.js`를 작성하고 `manifest.json`에 등록하면 코레일 페이지에 UI가 생기는데 — **실제로 어떤 과정을 거쳐서?**

### 1단계: 설치 = Chrome에 파일 경로 등록

`chrome://extensions`에서 "압축해제된 확장 프로그램 로드"를 하면 Chrome은 해당 폴더를 **확장 프로그램 패키지**로 인식하고 고유 ID를 부여한다. 이 ID는 `chrome-extension://[id]/` 형태의 전용 URL 스킴의 루트가 된다.

```
chrome-extension://abcdefghijklmnop/manifest.json
chrome-extension://abcdefghijklmnop/content.js
chrome-extension://abcdefghijklmnop/worker.js
```

확장 프로그램 파일들은 이 URL로만 접근 가능한 별도 공간에 존재한다. 일반 웹 페이지에서 `fetch('chrome-extension://...')`를 직접 날리면 차단된다.

### 2단계: URL 매칭 → 스크립트 주입

사용자가 `https://www.korail.com/ticket/...` 페이지를 열면, Chrome은 모든 설치된 확장 프로그램의 `manifest.json`을 순회하며 `content_scripts[].matches` 목록과 현재 URL을 대조한다. 패턴이 맞으면 페이지 로딩 직후 해당 JS/CSS를 그 탭에 **주입(inject)** 한다.

"주입"이 실제로 의미하는 것은, Chrome이 해당 탭의 렌더러 프로세스에 스크립트 실행 명령을 보내는 것이다. 개발자 도구 콘솔에서 JS를 직접 붙여넣고 실행하는 것과 메커니즘이 유사하다. 차이는 개발자 도구는 사람이 수동으로 트리거하지만, content script는 URL 조건이 맞을 때 Chrome이 자동으로 실행한다는 점이다.

`style.css`도 같은 방식으로 주입된다. Chrome이 페이지 `<head>`에 `<style>` 태그를 삽입하는 것과 동일한 효과다. 실제로 개발자 도구 Elements 탭을 보면 `<style>` 태그가 생긴 것을 확인할 수 있다.

### 3단계: Isolated World — 같은 DOM, 다른 JS 컨텍스트

여기서 핵심 개념이 하나 있다. Content script는 **페이지의 DOM은 공유하지만, JavaScript 실행 컨텍스트는 분리된다.**

```
┌─────────────────────────────────────────────────┐
│                  브라우저 탭                      │
│                                                 │
│  ┌───────────────────┐   ┌──────────────────┐  │
│  │  코레일 페이지 JS  │   │   content.js     │  │
│  │                   │   │   (내 코드)       │  │
│  │  window.pageVar   │   │  window.myVar    │  │
│  │  korail.func()    │   │  isMacroRunning  │  │
│  └────────┬──────────┘   └────────┬─────────┘  │
│           │    서로 접근 불가       │            │
│           └──────────┬────────────┘            │
│                      ↓                         │
│               공유 DOM (document)               │
│           document.querySelector(...)          │
│           element.click(), innerHTML 등         │
└─────────────────────────────────────────────────┘
```

- `content.js`에서 `document.querySelector('.btn')`을 쓰면 코레일 페이지의 버튼을 그대로 잡을 수 있다.
- 하지만 코레일 페이지 JS가 선언한 `window.someVar`나 함수는 `content.js`에서 직접 읽을 수 없다. JS 전역 스코프가 별개다.

이 격리(Isolated World) 덕분에 내 매크로 변수가 코레일 페이지 JS와 이름 충돌을 일으키지 않는다. 반대로, 코레일 JS가 실수로 내 변수를 덮어쓰는 일도 없다.

### 4단계: web_accessible_resources — 확장 파일을 페이지 컨텍스트에서 참조하기

Content script에서 Web Worker를 생성할 때 한 가지 문제가 생겼다.

```javascript
// 이렇게 쓰면 동작하지 않는다
const worker = new Worker('./worker.js');
// => korail.com/worker.js 를 찾으려 하므로 404
```

`new Worker()`에 상대 경로를 넘기면 현재 페이지(`korail.com`) 기준으로 파일을 찾는다. `chrome.runtime.getURL('worker.js')`를 쓰면 `chrome-extension://[id]/worker.js` 전체 경로를 반환해준다.

그런데 보안상 이유로 외부 페이지 컨텍스트에서 `chrome-extension://` 경로에 아무 파일이나 접근할 수는 없다. **`manifest.json`의 `web_accessible_resources`에 명시된 파일만** 허용한다.

```json
"web_accessible_resources": [
  {
    "resources": ["worker.js"],
    "matches": ["https://www.korail.com/*"]
  }
]
```

이 선언이 있어야 아래 코드가 정상 동작한다.

```javascript
const worker = new Worker(chrome.runtime.getURL('worker.js')); // ✅
```

`matches`를 `korail.com`으로 좁힌 이유도 있다. 다른 도메인 페이지에서 내 Worker 파일을 참조하는 걸 막기 위해서다.

### 정리: manifest.json 각 필드가 하는 일

```json
{
  "manifest_version": 3,
  "name": "코레일 예매 매크로",
  "permissions": ["storage"],
  "content_scripts": [
    {
      "matches": ["https://www.korail.com/ticket/*",
                  "https://www.korail.com/member/*"],
      "js": ["content.js"],
      "css": ["style.css"]
    }
  ],
  "web_accessible_resources": [
    {
      "resources": ["worker.js"],
      "matches": ["https://www.korail.com/*"]
    }
  ]
}
```

| 필드 | 역할 |
|------|------|
| `permissions` | Chrome API 사용 권한 선언. 설치 시 사용자에게 고지되는 항목 |
| `content_scripts.matches` | 스크립트를 주입할 URL 패턴. `/member/*`는 세션 만료 감지용 |
| `content_scripts.js/css` | 주입할 파일 목록. DOM 조작과 UI 스타일링 |
| `web_accessible_resources` | 페이지 컨텍스트에서 `chrome.runtime.getURL()`로 접근 허용할 파일 |

---

## 핵심 기술 문제와 해결

### 문제 1 — 백그라운드 탭 타이머 제한

가장 먼저 부딪힌 문제였다. Chrome은 백그라운드(숨겨진) 탭에서 `setTimeout` / `setInterval`을 **최소 1분으로 강제 제한**한다. 3초마다 새로고침해야 하는 매크로에 치명적이다.

**해결: Web Worker + Web Locks API 조합**

Web Worker는 별도 스레드에서 실행되므로 탭이 숨겨져도 타이머 제한을 받지 않는다.

```javascript
// worker.js — 500ms마다 메인 스레드에 tick 전송
setInterval(() => self.postMessage('tick'), 500);
```

```javascript
// content.js — Worker 메시지를 받아 예약된 작업 실행
function initWorker() {
    macroWorker = new Worker(chrome.runtime.getURL('worker.js'));
    macroWorker.onmessage = () => {
        if (macroPendingAction && Date.now() >= macroTargetTime) {
            const fn = macroPendingAction;
            macroPendingAction = null;
            fn();
        }
    };
}
```

여기에 **Web Locks API**를 추가해 Chrome의 intensive throttling도 우회했다.

```javascript
function holdWakeLock() {
    if (!navigator.locks) return;
    navigator.locks.request('ktx-macro-wake', { mode: 'shared' }, () => new Promise(() => {}));
}
```

`new Promise(() => {})` — 절대 resolve되지 않는 Promise로 락을 영구 점유한다. Chrome은 Web Lock을 보유한 탭을 절전 대상에서 제외한다.

---

### 문제 2 — 새로고침 후 상태 복원

예매 시도 → 페이지 reload → 매크로 상태 초기화. 이 사이클을 처리해야 했다.

페이지를 새로고침하면 content script도 처음부터 다시 실행된다. JS 메모리에 있던 변수는 전부 사라진다. 매크로가 실행 중이었다는 사실을 어딘가에 남겨야 한다.

**해결: sessionStorage로 상태 지속**

```javascript
// 새로고침 전 상태 저장
sessionStorage.setItem('isMacroRunningAfterReload', 'true');
sessionStorage.setItem('macroRefreshCount', refreshCount);

// 새로고침 후 복원
if (sessionStorage.getItem('isMacroRunningAfterReload') === 'true') {
    restoreMacroState();
}
```

`sessionStorage`는 탭이 살아있는 동안 유지되며, 새로고침해도 데이터가 남는다. 탭을 닫으면 사라진다. 매크로 설정값(새로고침 간격, 시간 구간 등)은 `localStorage`에 저장해 탭을 닫아도 유지했다.

---

### 문제 3 — 예매 버튼과 팝업 감지

코레일 페이지가 새로고침 없이 동적으로 DOM을 업데이트한다. 취소표가 생겼을 때 나타나는 예매 버튼을 어떻게 감지하느냐가 문제였다.

**해결: MutationObserver**

```javascript
const observer = new MutationObserver((mutations) => {
    for (const mutation of mutations) {
        for (const node of mutation.addedNodes) {
            if (node.nodeType === 1 && isBookingButton(node)) {
                clickBookingButton(node);
            }
        }
    }
});

observer.observe(document.body, { childList: true, subtree: true });
```

`MutationObserver`는 DOM 변화를 비동기로 감지한다. `setInterval`로 폴링하는 것보다 훨씬 반응이 빠르고, 불필요한 연산도 없다. 예매 중 나타나는 안내 팝업 자동 닫기에도 동일하게 적용했다.

---

### 문제 4 — 단순 반복 패턴 회피

일정한 간격으로 계속 요청을 보내면 패턴이 너무 규칙적이다.

**해결: ±40% 난수 지터**

```javascript
function getJitteredDelay(baseMs) {
    const jitter = (Math.random() * 0.8 - 0.4); // -40% ~ +40%
    return Math.round(baseMs * (1 + jitter));
}
```

3초 간격으로 설정하면 실제로는 1.8초~4.2초 사이에서 랜덤하게 동작한다.

---

## 기능 설계에서 신경 쓴 것들

### 우선순위 선택

체크박스를 클릭한 순서대로 번호 배지를 표시하고, 1순위부터 순서대로 예매를 시도한다. 일반석→창가석→통로석 순으로 선호한다면 그 순서대로 체크하면 된다.

### 시간 구간 스케줄링

취소표가 자주 풀리는 시간대가 있다. 예매 후 20분(결제 미완료 취소), 출발 3일 전, 당일 오전 9~10시, 출발 1시간 전. 이 시간대에만 빠른 간격으로 동작하고, 나머지 시간에는 5분 간격 절전 모드로 전환하도록 최대 5개 구간을 설정할 수 있게 만들었다. 자정을 넘는 구간(23:30~00:30)도 지원한다.

### 세션 만료 감지

코레일은 일정 시간 후 자동 로그아웃한다. 로그인 페이지로 리다이렉트되면 매크로가 조용히 멈춰버리는데, 이걸 사용자가 모르면 몇 시간을 허비할 수 있다. `/member/*` URL을 content_scripts 매칭에 추가해서 리다이렉트를 감지하고 즉시 데스크톱 알림을 보내도록 처리했다.

---

## 처음 Chrome 확장 프로그램을 만들면서 느낀 것

웹 개발 경험이 있어도 확장 프로그램은 다른 맥락이 있다. 처음엔 "그냥 JS 파일 하나 끼워 넣는 거 아닌가?" 싶었는데, Isolated World 개념, `chrome-extension://` URL 스킴, web_accessible_resources 접근 제어 같은 것들이 하나하나 이유가 있었다.

가장 인상 깊었던 건 **브라우저가 생각보다 훨씬 많은 것을 제한한다**는 점이다. 백그라운드 탭 타이머 제한, intensive throttling, Web Worker의 별도 글로벌 컨텍스트 등 — 일반 웹 개발에서는 신경 쓰지 않았던 제약들이 여기서는 핵심 문제가 됐다.

Manifest V3로 넘어오면서 background service worker가 ephemeral(비영속)하게 바뀐 것도 설계에 영향을 줬다. 상태를 메모리에 두지 않고 storage에 명시적으로 저장해야 한다는 원칙이 강제된다.

구조 자체는 단순하다. `manifest.json` + `content.js` + `worker.js` 세 파일이 전부다. 복잡한 빌드 도구도 없고, npm도 없다. 오히려 그 단순함 덕분에 브라우저 API에 집중할 수 있었다.
