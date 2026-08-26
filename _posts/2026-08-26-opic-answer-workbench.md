---
layout: post
title: "OPIc 시뮬레이터 — 시험 재현기에서 학습 작업대로"
date: 2026-08-26
categories: [개발, 회고]
tags: [OPIc, GPT-4o, 프롬프트엔지니어링, Vue, WebSpeechAPI, UX]
---

## 오늘 한 일

- 문항 1:1 `내 답안` 을 세 화면(문제 은행 · 시험 설정 · Practice)이 공유하게
- AI 답안 코치 — 대충 적은 초안을 말할 수 있는 스크립트로
- 받아적기(Web Speech API) — AI 아님, 무과금
- 문제 은행에 콤보별 보기 + Set 1·2·3 추가
- `canServe` 가 콤보의 첫 유형만 검사하던 문제 수정

## 핵심 작업 해설

### 저장소를 하나로 두면 이름이 갈리지 않는다

`내 답안` 을 세 화면에 붙이면서 가장 신경 쓴 건 **키 규칙**이었다.

```javascript
function keyForRel(rel) {
  rel = decodeURIComponent(rel).replace(/^.*?questions\//, '');
  return rel ? 'a:' + rel : '';
}
```

음원 상대경로를 키로 쓴다. 시험 설정 미리보기는 `Mediafile` 전체 경로를, 문제 은행은
`rel` 경로를 갖고 있는데, 같은 함수를 거치면 같은 키가 나온다. 그래서 어디서 써도
나머지 두 곳에서 같이 보인다. 회귀 테스트가 이 계약을 직접 검증한다 —
미리보기에서 쓰고 → 문제 은행에서 수정 → Practice에서 확인.

처음엔 '준비한 답안'(편집)과 '실제로 말한 답변'(기록)을 다른 탭으로 갈라 뒀다가
하나로 합쳤다. 둘 다 "내 답변"이라 이름이 겹쳤고, 무엇보다 **준비한 것과 말한 것을
나란히 비교할 수가 없었다.** 지금은 편집기가 위, 지난 응시 기록이 아래에 접혀 있다.

### 목표 길이가 창작 금지를 이겨 버렸다

AI 코치의 첫 결과물은 이랬다. 입력은 "이름 상일, 컴퓨터 전공, 작년 졸업, 취업 준비중,
자취" 다섯 줄.

> I've been **on the hunt for** a job... Samsung because it's such a **prominent**
> company... I get to enjoy **my mom's cooking**... **All in all**, I'm excited about
> the opportunities that **lie ahead**.

문제가 둘이었다. 표현이 너무 어려워서 단기간에 입에 안 붙고, 초안에 없는 삼성·엄마
요리·공과금이 들어왔다.

프롬프트에 "지어내지 마라"를 이미 써 뒀는데도 그랬다. 원인은 **지시 충돌**이었다.
5단계 목표 길이 160~200단어를 함께 줬으니, 다섯 줄로 그 숫자를 맞추려면 지어낼 수밖에
없다. 길이 목표를 종속시켰다.

```python
aim = (
    f"Length GOAL: about {w_lo}~{w_hi} words (level {level}).\n"
    "!! The goal is secondary. RULE 1 (do not invent) always wins. !!\n"
    f"The notes hold roughly {draft_words} pieces of information. If that cannot honestly\n"
    f"fill {w_lo} words, write a SHORTER script and stop.\n"
    "You may end with ONE sentence containing [브라켓] blanks the learner can fill in.\n"
    "A truthful 70-word script beats an invented 170-word one.")
```

같은 입력으로 다시 돌리니 127단어에서 멈추고, 없는 얘기 대신 `[special field]` 같은
빈칸을 남기고, 무엇을 채우면 되는지 한국어로 알려 줬다.

난이도는 나쁜 예/좋은 예를 직접 넣어 잡았다.

```
I've been on the hunt for a job.         -> I'm looking for a job.
It's such a prominent company.           -> It's a really big company.
All in all, I'm excited about my future. -> So, I'm excited about the future.
```

`polish` 모드에도 같은 함정이 있었다. 등급별 목표를 함께 주니 32단어 초안이 180단어로
불어났다. 초안 자신의 길이를 기준으로 바꿔 33단어가 됐다.

### 저장 형식이 곧 표시 형식

문단과 강조를 "AI 제안에서만" 보여 주면 저장하는 순간 사라진다. 저장 형식 자체를
평문 규칙으로 정했다 — 빈 줄로 문단, `**...**` 로 핵심 표현.

```javascript
renderHtml: function (text) {
  var paras = esc.split(/\n\s*\n/);          // 빈 줄 = 문단
  body = body.replace(/\*\*([^*\n]+?)\*\*/g, '<b class="nt-key">$1</b>');
  body = body.replace(/\[([^\]\n]+)\]/g, '<span class="nt-blank">[$1]</span>');
}
```

편집칸은 원본(별표 포함)을, 보기는 렌더링을 보여 준다. 세 화면이 같은 함수를 쓰므로
"어디선 굵기가 되고 어디선 안 되는" 상태가 생기지 않는다.

## 문제 & 해결

### 고를 수 있는데 골라도 안 되는 주제

문제 은행에 콤보별 보기를 붙이다가 이상한 걸 발견했다. `건강` 은 T1·T2·T3만 있고
T4가 없는데, `T1·T3·T4` 를 요구하는 콤보 II·III의 후보로 잡혔다.

```javascript
// 예전 canServe — require 가 없으면 첫 유형만 봤다
var first = combo.types[0];
for (var j = 0; j < alts.length; j++) {
  if (hasType(topic, alts[j])) return true;   // T1 만 보고 통과
}
```

증상이 조용했다. 200회 무작위 출제에서는 백트래킹이 막아 문제가 없었다. 그런데
시험 설정에서 콤보 II에 `건강` 을 직접 고정하면 **요리가 나온다.** 드롭다운이
불가능한 조합을 제안하고, 골라도 아무 말 없이 다른 주제로 바뀌었다.

모든 슬롯을 채울 수 있어야 후보가 되게 고쳤다. 5~6단계 콤보 II·III의 돌발 후보가
18→5개로 줄지만, 출제공식 검증(난이도별 200회)은 그대로 통과했다.

### Set이라는 개념이 빠져 있었다

교재는 한 주제에 유형별로 여러 문항을 둔다. `가족` 은 T1~T4가 각 3개다. 그 N번째끼리
모으면 실제로 한 세트로 출제되는 조합이 된다.

Set 수를 유형별 문항 수의 **최대값**으로 잡으면 `공원` 처럼 T7만 2개인 주제에서
대부분 빈 Set2가 생긴다. **최소값**으로 바꿨다 — 그래야 Set이 "실제로 받을 수 있는
완성된 조합"을 뜻한다.

## 배운 점

- **프롬프트에 규칙을 쓰는 것과 규칙이 이기게 만드는 건 다르다.** "지어내지 마라"와
  "160단어를 채워라"를 나란히 두면 후자가 이긴다. 우선순위를 명시해야 한다.
- 학습 도구에서 AI 출력은 **내가 쓴 것과 구별돼야** 한다. 초안을 자동으로 덮어쓰지
  않고, 손댄 곳을 공개하고, 넣어도 저장은 따로 누르게 했다.
- 후보 판정이 관대하면 UI가 거짓말을 한다. 고를 수 있게 보여 줬으면 골랐을 때
  그대로 돼야 한다.

## 다음 작업

- localStorage 내구성 — 내 답안·프로필·⭐이 브라우저 기록에만 있다
