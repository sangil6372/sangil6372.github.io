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

## Chrome 확장 프로그램 구조 — 처음 알게 된 것들

### Manifest V3

현재 Chrome 확장 프로그램의 표준 스펙이다. `manifest.json`이 모든 것의 시작점이다.

```json
{
  "manifest_version": 3,
  "name": "코레일 예매 매크로",
  "permissions": ["storage"],
  "content_scripts": [
    {
      "matches": ["https://www.korail.com/ticket/*"],
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

**content_scripts** — 지정한 URL 패턴에 해당하는 페이지가 열릴 때 자동으로 주입되는 JS/CSS다. 일반 웹 페이지 DOM에 접근할 수 있다. 이 프로젝트에서는 `content.js` 하나가 UI 렌더링부터 예매 자동화까지 전부 담당한다.

**web_accessible_resources** — content script에서 `chrome.runtime.getURL()`로 확장 프로그램 내부 파일에 접근하려면 여기에 명시해야 한다. `worker.js`를 Web Worker로 로드할 때 이게 필요하다.

**permissions** — 최소 권한 원칙. `storage`만 선언했다. `notifications`는 Manifest V3에서 content script에서 직접 `new Notification()`으로 사용할 수 있어서 별도 권한 선언이 불필요했다.

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

3초 간격으로 설정하면 실제로는 1.8초~4.2초 사이에서 랜덤하게 동작한다. 서버 입장에서 사람의 행동과 구별하기 어렵게 만드는 의도다.

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

웹 개발 경험이 있어도 확장 프로그램은 다른 맥락이 있다. 페이지 JS가 아니라 브라우저 위에서 동작하는 코드를 쓰는 느낌이랄까.

가장 인상 깊었던 건 **브라우저가 생각보다 훨씬 많은 것을 제한한다**는 점이다. 백그라운드 탭 타이머 제한, intensive throttling, Web Worker의 별도 글로벌 컨텍스트 등 — 일반 웹 개발에서는 신경 쓰지 않았던 제약들이 여기서는 핵심 문제가 됐다.

Manifest V3로 넘어오면서 background service worker가 ephemeral(비영속)하게 바뀐 것도 설계에 영향을 줬다. 상태를 메모리에 두지 않고 storage에 명시적으로 저장해야 한다는 원칙이 강제된다.

구조 자체는 단순하다. `manifest.json` + `content.js` + `worker.js` 세 파일이 전부다. 복잡한 빌드 도구도 없고, npm도 없다. 오히려 그 단순함 덕분에 브라우저 API에 집중할 수 있었다.
