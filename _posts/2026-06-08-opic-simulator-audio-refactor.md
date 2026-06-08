---
layout: post
title: "OPIc 시뮬레이터 음원 구조 통합과 T9/T10 TTS 추가"
date: 2026-06-08
categories: [개발, 회고]
tags: [OPIc, EdgeTTS, Python, 음성처리, 리팩토링]
---

## 오늘 한 일

- `questions/original/` 음원 322개를 `custom/` 명명 규칙으로 일괄 복사·정리
- `questions.json`, `transcripts.json` 경로 전면 갱신
- Edge TTS로 T9/T10 음성 12개 신규 생성 (6개 주제 × 비교·이슈 각 1개)
- 세션 빌더의 Combo V 완성 → 시험 세션 13문제 → 15문제로 확장

---

## 핵심 작업 해설

### 1. 음원 구조 통합 — `migrate_to_custom.py`

기존에 `questions/original/` 아래 음원 파일명은 교재 메타데이터를 그대로 포함하고 있었다.

```
Unit 03 독신 1-1 Q page 68.mp3
Unit 51 호텔IRP 7-1 Q page 394.mp3
```

이 상태로는 파일명만 보고 유형을 알 수 없고, 롤플레잉 서브폴더(`롤플레잉/`)까지 섞여 있어 관리가 어려웠다. 목표는 `custom/` 폴더 기준의 통일된 패턴으로 변환하는 것이었다.

```
독신_T1_1_Q.mp3
호텔_T7_1_Q.mp3   ← IRP/IIRP 구분 없이 variant 순번으로 통합
```

핵심 로직은 `questions.json`을 정답지로 삼아 파일을 복사하는 방식이다. 파일명 파싱 없이 JSON의 배열 인덱스를 variant 번호로 그대로 사용했다.

```python
for type_key, file_list in topic['q'].items():
    for idx, old_rel in enumerate(file_list):
        variant  = idx + 1
        new_name = f'{name}_T{type_key}_{variant}_Q.mp3'
        shutil.copy2(
            os.path.join(ORIGINAL_BASE, old_rel),
            os.path.join(CUSTOM_BASE, name, new_name)
        )
        path_map[old_rel] = f'{name}/{new_name}'
```

스크립트 실행 한 번으로 322개 복사, JSON 두 개 갱신, 구형 파일 4개 삭제까지 한 번에 처리했다.

### 2. T9/T10 추가 — Combo V 15문제 완성

세션 빌더(`session-builder.js`)는 15문제를 5개 Combo로 구성한다.

| Combo | 문제 | 유형 |
|-------|------|------|
| I | Q2~Q4 | T1·T2·T3 (묘사·루틴·경험) |
| II | Q5~Q7 | T1·T3·T4 |
| III | Q8~Q10 | T1·T3·T4 (돌발 주제) |
| IV | Q11~Q13 | T6·T7·T8 (롤플레이) |
| **V** | **Q14~Q15** | **T9·T10** |

Combo V는 `questions.json`의 `custom` 섹션에서 T9(과거/현재 비교)·T10(사회 이슈) 음성을 찾는다. original 649개 파일을 전수 조사했더니 T9/T10은 단 한 개도 없었다. 미매핑 327개는 전부 예시답안 녹음이었다.

`generate_mp3.py`에 `t9`/`t10` 필드 처리를 추가하고, `questions_input.json`에 6개 주제의 질문 텍스트를 작성했다.

```python
# generate_mp3.py 추가 로직
for type_key in ("t9", "t10"):
    text = topic.get(type_key)
    if not text:
        continue
    filename = f"{topic_id}_T{type_key[1:]}_1_Q.mp3"
    await generate_one(text, os.path.join(folder_path, filename))
```

Edge TTS(`en-US-JennyNeural`, `-5%` 속도)로 12개 파일을 생성했다. 질문 방향은 다음과 같다.

- **T9** (비교/변화): "How has [topic] changed over the past 10 years..."
- **T10** (이슈): "What do you think are the most serious problems related to [topic]..."

---

## 문제 & 해결

**`questions.json`의 `custom` 섹션 경로 구조가 `selected`와 다르다.**

`selected`/`surprise`/`roleplayOnly` 섹션의 경로는 `base`(`../questions/custom/`)를 기준으로 하는 상대 경로다. 그런데 `custom` 섹션은 `customBase`(`../questions/`)를 기준으로 하기 때문에 경로에 `custom/` 프리픽스가 붙어야 한다.

```json
// selected 경로: base + 이 값
"1": ["독신/독신_T1_1_Q.mp3"]

// custom 경로: customBase + 이 값
"9": ["custom/음악/음악_T9_1_Q.mp3"]
```

`session-builder.js` 코드를 추적해서 `pickFileFull`과 `pickFileFullCustom` 두 함수가 각각 다른 base를 쓴다는 걸 확인하고 나서야 해결됐다.

---

## 배운 점

- JSON을 정답지로 삼아 스크립트를 짜면 파일명 파싱 없이 안전하게 대량 복사가 가능하다.
- Edge TTS는 tts_server 없이 `edge_tts` 라이브러리 직접 호출로 스탠드얼론 실행이 가능하다.
- `migrate_to_custom.py` 같은 일회성 마이그레이션 스크립트도 `--dry-run` 옵션을 넣어두면 검증이 훨씬 편하다.

---

## 다음 작업

- 세션 중 T5(상대방에게 질문하기) 유형이 사용되지 않는 문제 해결
- `questions_input.json` 돌발 경로 오류(`custom/돌발/가구` → `custom/가구`) 정리
- 요가·캠핑 등 original 없는 주제 TTS 음원 생성 및 등록
