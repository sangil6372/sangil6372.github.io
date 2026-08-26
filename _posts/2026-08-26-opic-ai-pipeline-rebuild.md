---
layout: post
title: "OPIc 시뮬레이터 — 발음 분석 AI를 걷어내고 측정값으로 바꾼 이유"
date: 2026-08-26
categories: [개발, 회고]
tags: [OPIc, GPT-4o, whisper, 재현성, 비용최적화, aiohttp]
---

## 오늘 한 일

- `gpt-audio` 기반 발음 분석 제거 — 재현성 없음을 실측으로 확인
- 전사를 `whisper-1` 로 이관, 분석 단계 17초 → 3초
- 전달력 지표(속도·멈춤·필러·되풀이)를 단어 타임스탬프에서 직접 계산
- 동기 OpenAI 클라이언트를 `AsyncOpenAI` 로 교체 — 이벤트 루프 블로킹 해소
- 과금 엔드포인트 전부 캐시 + 결과 페이지 재과금 제거

## 핵심 작업 해설

### 발음 분석은 "듣지" 않고 있었다

`/evaluate-feedback` 은 `gpt-audio` 에게 녹음을 주고 전사와 발음 이슈를 함께 받았다.
그럴듯해 보였지만 같은 녹음으로 5회 돌려 보니 결과가 이랬다.

```
1회  ['connectedness', 'stress']
2회  ['articulation', 'intonation']
3회  ['intonation', 'connectedness']
4회  ['intonation', 'articulation']
5회  <JSON 파싱 실패>
→ 서로 다른 결과 조합 4/5
```

결정적인 근거는 다른 데 있었다. **Edge TTS로 만든 합성음**(발음이 완벽한 기계 음성)을
넣었는데도 `articulation`, `intonation` 문제를 지적했다. 들은 게 아니라 지어낸 값이다.

회차마다 라벨이 바뀌면 학습자는 발음이 나아졌는지 알 수 없다. 게다가 문항당 $0.013로
가장 비싼 구성요소였다. 15문항 한 번 응시에 $0.20.

### modalities 함정

전사만 따로 받으려고 `modalities` 에서 `'audio'` 를 빼 봤다. 그러자 응답이 이랬다.

```
"Please provide the speech sample so I can listen and evaluate..."
```

그런데 usage 를 보면 `prompt_tokens_details.audio_tokens = 108`. **음성은 받았는데
못 들은 척한다.** 반대로 `text+audio` 로 두면 답을 음성으로도 만들어서, 전사까지
시키면 그 긴 텍스트를 통째로 낭독하느라 17초/2000토큰이 나가고 `finish_reason='length'`
로 JSON이 잘렸다.

결국 전사는 `whisper-1` 이 맡는 게 맞았다.

```python
tr_res = await client.audio.transcriptions.create(
    model='whisper-1',
    file=('answer.mp3', mp3_bytes, 'audio/mpeg'),
    language='en',
    response_format='verbose_json',
    timestamp_granularities=['word'],   # 이게 핵심
)
```

2.5초에 끝나고, **단어별 타임스탬프**까지 준다.

### 측정할 수 있는 것만 근거로 쓴다

타임스탬프가 있으면 LLM에게 물어볼 필요가 없는 것들이 있다.

```python
for w in words:
    speaking += max(0.0, w.end - w.start)
    if prev_end is not None and w.start - prev_end >= PAUSE_MIN:
        gaps.append(round(w.start - prev_end, 1))
    prev_end = w.end
```

속도·멈춤 횟수·최장 멈춤·발화 비율·필러·즉시 되풀이가 나온다. **무과금이고 값이
흔들리지 않는다.** 이 값을 피드백 프롬프트의 근거로 넣으니 "말이 빠른 편입니다" 같은
막연한 코멘트가 "분당 176단어로 다소 빠르며 0.5초 이상 멈춤이 2회" 로 바뀌었다.

Diagnostic Form의 발음 항목도 손봤다. 근거 없이 채우지 말라고 프롬프트에 못박았다.

## 문제 & 해결

### 이벤트 루프가 막혀 있었다

async 핸들러 안에서 동기 `OpenAI()` 와 `subprocess.run` 을 쓰고 있었다. 클라이언트가
아무리 병렬로 요청을 보내도 서버가 한 건씩만 처리한다는 뜻이다.

`AsyncOpenAI` + `asyncio.create_subprocess_exec` 로 바꾸고 측정했더니, 서로 다른 3건
동시 발사가 직렬 32.9초 대신 **11.2초**에 끝났다.

### 결과를 다시 보는데 왜 돈이 나가지

`result.html` 의 `onMounted` 가 조건 없이 `/overall-feedback` 을 불렀다. 결과 페이지를
세 번 열면 15문항 분석이 세 번 청구된다. 받은 분석을 세션에 저장해 재사용하고,
재생성은 **↻ 다시 분석** 버튼으로만 하게 했다. 회귀 테스트가 호출 횟수를 센다.

```
✅ 첫 방문에 전체 분석 1회
✅ 재방문은 저장된 분석 재사용 (재과금 없음)  1 → 1회
✅ 다시 분석 버튼은 재호출  2회
```

## 배운 점

- **"그럴듯한 출력"과 "믿을 수 있는 출력"은 다르다.** 발음 라벨은 형식이 완벽했고
  JSON도 유효했다. 같은 입력으로 5번 돌려 보기 전까지는 문제를 몰랐다.
- 합성음처럼 **정답을 아는 입력**을 하나 끼워 두면 모델이 지어내는지 바로 드러난다.
- LLM에 물어보기 전에 계산할 수 있는지 먼저 확인할 것. 계산값은 공짜고 재현된다.

## 다음 작업

- localStorage 내구성 — 브라우저 기록을 지우면 학습 자료가 날아간다
