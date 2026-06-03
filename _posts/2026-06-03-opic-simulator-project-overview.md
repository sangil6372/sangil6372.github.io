---
layout: post
title: "OPIc 시뮬레이터 프로젝트 기록 — 구조 설계부터 AI 피드백까지"
date: 2026-06-03
categories: [프로젝트, 기록]
tags: [opic, gpt-4o, edge-tts, python, aiohttp, javascript, web-speech-api, github-pages]
---

## 프로젝트 개요

OPIc(Oral Proficiency Interview - computer) 시험 환경을 로컬에서 재현하고, 녹음된 답변에 AI 피드백을 제공하는 시뮬레이터.

**핵심 기능 3가지:**
- 실제 OPIc 시험과 동일한 UI로 시험 진행
- Edge TTS로 영어 문제 음성 자동 생성
- GPT-4o 기반 OPIc 등급 채점(NH~AH) + 문항별 상세 피드백

**기술 스택:**

| 영역 | 기술 |
|------|------|
| 프론트엔드 | HTML/CSS/JS (Vue.js 일부), Web Speech API |
| 백엔드 | Python, aiohttp (port 8765) |
| TTS | Microsoft Edge TTS (`edge-tts`) |
| AI | OpenAI GPT-4o |
| 배포 | Vercel (정적 프론트), 로컬 Python 서버 (TTS/AI) |

---

## 개발 타임라인

### Phase 1 — 페이지 구조 파악 및 클로닝 (5/25)

OPIc 공식 데모 사이트의 HTML/CSS/JS를 분석해서 시험 흐름을 역설계했다.

```
Background Survey → Self Assessment → Pre-Test Setup
→ Begin Test → Sample Question → OPIc Test (본시험)
```

각 페이지를 로컬에 복제하고 외부 의존성(CDN 등)을 로컬 assets로 교체했다. 원본 파일은 `pages/originals/`에 보존.

---

### Phase 2 — 세션 빌더 구현 (5/25~26)

`src/session-builder.js`에서 `questions.json`을 읽어 시험 세션을 동적으로 구성한다.

**questions.json 구조:**
```json
{
  "intro": { "name": "자기소개", "q": { "1": ["파일경로.mp3"] } },
  "selected": [ { "name": "공원", "q": { "1": [...], "2": [...] } } ],
  "surprise": [ ... ],
  "custom":   [ ... ]
}
```

OPIc 실제 문제 구성 방식(intro 1문항 + selected 4개 주제 + surprise 2개)을 `buildSession()`으로 재현. combo 번호(T1~T8: 일반, T6~T8: 롤플레이, T9~T10: 돌발)를 기반으로 문항 유형을 분류한다.

---

### Phase 3 — TTS 문제 음성 생성 (5/27)

`generate_mp3.py` — `questions_input.json`의 텍스트를 읽어 Edge TTS로 MP3 일괄 생성.

```python
comm = edge_tts.Communicate(text, voice='en-US-JennyNeural', rate='-5%')
await comm.save(output_path)
```

생성된 파일은 `questions/custom/{주제명}/` 에 저장. 파일명 규칙:
```
{주제명}_T{qtype}_{index}_Q.mp3
```

---

### Phase 4 — 트랜스크립트 및 전처리 (5/31)

**`generate_transcripts.py`** — 문제 텍스트를 `transcripts.json`으로 추출. admin 페이지에서 문제 원문을 미리보기 하는 데 사용.

**`trim_intro_numbers.py`** — 음성 파일 앞에 "1.", "2." 같은 문항 번호 읽는 부분을 제거. OPIc 실제 시험에서는 번호를 읽지 않기 때문.

---

### Phase 5 — TTS 서버 + 어드민 + 결과 페이지 (6/1~2)

#### `tts_server.py` (aiohttp, port 8765)

주요 엔드포인트:

| 엔드포인트 | 역할 |
|-----------|------|
| `POST /tts` | Edge TTS MP3 생성 |
| `GET /scan-themes` | custom 폴더 문제 목록 조회 |
| `POST /feedback` | 단일 문항 GPT-4o 피드백 |
| `POST /overall-feedback` | 전체 시험 종합 채점 |
| `POST /save-recordings` | 녹음 파일 저장 |
| `GET /apikey-status` | OpenAI API 키 확인 |

#### `public/admin.html`

주제별 TTS 문제 생성 및 관리 UI. Vue.js 기반 SPA로 구현.
- 주제 선택 → 문항별 텍스트 편집 → TTS 생성 → 미리듣기
- questions.json 자동 업데이트 (`/update-qjson`)

#### `pages/result.html`

시험 완료 후 GPT-4o 분석 결과 표시.
- localStorage의 `opic_session_result`에서 세션 로드
- `/overall-feedback` 호출 → 최종 등급(NH~AH) + 강점/개선점/다음등급 조언
- 문항별 피드백 아코디언 UI

---

### Phase 6 — 피드백 팝업 UI 전면 개선 (6/3)

시험 중 "💬 내 답변 듣기 + AI 피드백" 버튼으로 열리는 팝업을 전면 재설계.

#### 기존 문제점
1. AI 피드백이 `white-space: pre-wrap` 텍스트 덩어리
2. 모범 답변이 수험자 원문을 거의 그대로 다듬는 수준
3. 녹음 재생 = `new Audio(url).play()` 단 한 줄, 컨트롤 없음
4. 팝업 너비 660px으로 좁음

#### 개선 1 — 섹션별 카드 렌더링

`##` 헤더로 피드백을 파싱해 섹션마다 다른 색상 카드로 렌더링:

```javascript
function _renderFeedback(raw) {
    var parts = esc.split(/(?=^## )/m);
    parts.forEach(function(part) {
        if (title.indexOf('📊') > -1) cls = 'diagnosis'; // 황색
        if (title.indexOf('🔍') > -1) cls = 'analysis';  // 청색
        if (title.indexOf('⬆')  > -1) cls = 'tips';      // 녹색
        if (title.indexOf('💎') > -1) cls = 'model';     // 보라 → 오른쪽 컬럼
    });
}
```

#### 개선 2 — 2컬럼 레이아웃

```css
.fb-box    { width: 94vw; max-width: 1280px; }
.fb-columns { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; }
```

왼쪽: 질문 + 내 답변 + 분석 피드백 / 오른쪽: 모범 답변 즉시 표시

#### 개선 3 — 오디오 플레이어

```
[▶/⏸]  ━━━━●━━━━━━━━━  0:32 / 1:19  [↩]
```

- `timeupdate` 이벤트로 fill 너비 + thumb 위치 동기화
- `mousedown` → `document.mousemove` → `mouseup` 드래그 시크
- document에 이벤트를 걸어야 트랙 밖으로 마우스가 나가도 드래그 유지됨

#### 개선 4 — STT 정제 파이프라인

```
음성인식(Web Speech API) → GPT-4o 정제 → GPT-4o 피드백 생성
```

정제 프롬프트: *"음성인식 오류만 교정, 내용·수준은 유지"*

수험자가 말한 영어 수준을 올려버리면 피드백이 왜곡되기 때문에 내용 변경 없이 오인식만 수정한다.

#### 개선 5 — 모범 답변 프롬프트 강화

```
- 주제(topic)만 유지하고 수험자의 문장 구조·표현은 버릴 것
- 150단어 이상
- 관계절 1개 이상, 부사절/분사구문 1개 이상
- 고급 어휘 5개 이상 업그레이드 필수
```

"핵심 내용을 살리되" → "표현은 버릴 것" 으로 바꾸자 GPT가 완전히 다른 수준의 답변을 생성하기 시작했다.

---

## 현재 구조 요약

```
opic_simulator/
├── tts_server.py          # aiohttp 서버 (port 8765)
├── generate_mp3.py        # TTS 일괄 생성
├── generate_transcripts.py
├── trim_intro_numbers.py
├── public/
│   ├── index.html         # 시작 화면
│   └── admin.html         # 문제 관리 어드민
├── pages/
│   ├── opic-test.html     # 본시험 + 피드백 팝업
│   └── result.html        # AI 채점 결과
├── src/
│   ├── session-builder.js
│   └── data/
│       ├── questions.json
│       └── transcripts.json
└── questions/
    ├── original/          # 원본 MP3 (git 제외)
    └── custom/            # 생성된 MP3 (git 제외)
```

---

## 다음 작업

- [ ] IH 수준 / AL 수준 모범 답변 분리 표시
- [ ] 모범 답변 한글 해설 추가
- [ ] result.html 피드백 UI도 동일한 카드 스타일로 통일
- [ ] Vercel 배포 후 프론트 퍼블릭 테스트
