---
layout: post
title: "OPIc 시뮬레이터 - 피드백 팝업 UI 개선 & 자동화 파이프라인 구축"
date: 2026-06-03
categories: [개발, 회고]
tags: [opic, gpt-4o, edge-tts, javascript, python, github-pages]
---

## 오늘 한 일

오늘은 OPIc 시뮬레이터 프로젝트를 GitHub에 올리고, 피드백 팝업 UI를 대폭 개선했다.

---

## 1. GitHub 버전 관리 시작

로컬에만 있던 프로젝트를 `sangil6372/vibe` 레포에 올렸다.

`.gitignore` 설정이 핵심이었다. MP3 파일이 854개나 있어서 모두 제외했다. Git에 바이너리 대용량 파일을 넣으면 레포가 무거워지고 GitHub의 100MB 단일 파일 제한에도 걸린다. TTS로 생성 가능한 파일은 커밋할 이유가 없다.

```
questions/custom/**/*.mp3
questions/original/**/*.mp3
recordings/
.env
```

Conventional Commits 컨벤션도 도입했다.

| 타입 | 용도 |
|------|------|
| `feat:` | 새 기능 |
| `fix:` | 버그 수정 |
| `refactor:` | 로직 변경 없는 코드 개선 |
| `chore:` | 빌드·설정 |
| `docs:` | 문서 |

---

## 2. 피드백 팝업 UI 전면 개선

### 문제점

기존 피드백 팝업은 세 가지 문제가 있었다.

1. **AI 피드백이 텍스트 덩어리** — `white-space: pre-wrap`으로 줄바꿈만 유지. `##` 섹션 헤더가 `<strong>` 태그로만 변환되어 시각적 구분이 없었다.
2. **모범 답변이 원문을 그대로 다듬는 수준** — GPT 프롬프트가 "핵심 내용을 살리되 AL 수준으로 완성"이라고 했더니 너무 원문에 의존했다.
3. **녹음 재생에 컨트롤이 없었다** — `new Audio(url).play()` 한 줄이 전부. 되감기, 일시정지가 불가능했다.

### 해결: 섹션별 카드 렌더링

`##`으로 구분된 피드백 텍스트를 파싱해서 섹션마다 다른 색상 카드로 렌더링했다.

```javascript
function _renderFeedback(raw) {
    var parts = esc.split(/(?=^## )/m);
    parts.forEach(function(part) {
        var cls = 'generic';
        if (title.indexOf('📊') > -1)      cls = 'diagnosis'; // 황색
        else if (title.indexOf('🔍') > -1) cls = 'analysis';  // 청색
        else if (title.indexOf('⬆') > -1)  cls = 'tips';      // 녹색
        else if (title.indexOf('💎') > -1) cls = 'model';     // 보라
        // model 섹션은 오른쪽 컬럼에 분리
        if (cls === 'model') modelHtml = content;
        else mainHtml += ...
    });
}
```

팝업을 2컬럼 레이아웃(`grid-template-columns: 1fr 1fr`)으로 바꿔서 왼쪽에 분석, 오른쪽에 모범 답변이 바로 보이도록 했다.

### 해결: 오디오 플레이어

`new Audio(url)` 기반 커스텀 플레이어를 만들었다.

```
[▶/⏸] [━━━━●━━━━━━━━━] [0:32 / 1:19] [↩]
```

시크바는 `mousedown` + `document.mousemove` + `mouseup` 조합으로 드래그를 구현했다. 썸(●)은 `position: absolute; left: {pct}%`로 fill과 같은 퍼센트로 이동시킨다.

```javascript
audio.addEventListener('timeupdate', function() {
    var pct = (audio.currentTime / audio.duration * 100);
    $('#fbSeekFill').css('width', pct + '%');
    $('#fbSeekThumb').css('left', pct + '%');
});
```

### 해결: 모범 답변 프롬프트 강화

```
## 💎 AL 수준 모범 답변 (완전 재작성)
- 주제(topic)만 유지하고 수험자의 문장 구조·표현은 버릴 것
- 150단어 이상
- 관계절 1개 이상, 부사절/분사구문 1개 이상
- 고급 어휘 5개 이상 업그레이드 필수
```

"핵심 내용을 살리되"라는 표현이 GPT가 원문을 따라가도록 유도하고 있었다. 이걸 "표현은 버릴 것"으로 바꾸자 완전히 다른 답변이 나왔다.

---

## 3. STT 정제 파이프라인

Web Speech API 음성인식 결과를 그대로 GPT에 넘기면 오인식된 단어가 피드백에 영향을 준다. 그래서 피드백 생성 전 1단계를 추가했다.

```
음성인식(STT) → GPT-4o 정제 → GPT-4o 피드백 생성
```

정제 프롬프트는 단순하다: *"음성인식 오류만 교정, 내용·수준은 그대로 유지"*. 수험자가 말한 영어 수준을 올려버리면 피드백이 왜곡된다.

---

## 배운 점

- CSS `position: sticky`는 스크롤 컨테이너 안에서만 동작한다. `overflow-y: auto`인 부모 안에 있어야 제대로 작동하는데 팝업이 `overflow-y: auto`라 모범답변 컬럼이 스크롤 따라 고정된다.
- GPT 프롬프트에서 **금지 표현**이 **허용 표현**보다 훨씬 강력하다. "살리되"보다 "버릴 것"이 더 명확한 지시다.
- 드래그 시크바는 `mousedown`을 트랙에 걸고 `mousemove`/`mouseup`은 `document`에 걸어야 한다. 트랙 밖으로 마우스가 나가도 드래그가 유지된다.

---

## 다음 작업

- IH 수준 / AL 수준 모범 답변 분리 표시
- 모범 답변 한글 해설 추가
- GitHub Pages 블로그 자동화 스케줄링
