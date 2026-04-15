---
name: cardnews-orchestrator
description: "카드뉴스 자동화 파이프라인 오케스트레이터. 페르소나+욕구 입력으로 카드뉴스 5장 세트를 자동 생성한다. 카드뉴스 만들어줘, 카드뉴스 생성, 썸네일 생성, 본문 작성, 이미지 생성 요청 시 사용. 후속: 카드뉴스 다시 만들기, 썸네일만 재생성, 본문 수정, 이미지 재생성, 이전 결과 개선, 다른 페르소나로, 앵글 바꿔줘."
---

# Cardnews Orchestrator — 카드뉴스 자동화 파이프라인

## 실행 모드: 서브 에이전트 (파이프라인 패턴)

## 에이전트 구성

| 에이전트 | 역할 | 스킬 | 출력 |
|---------|------|------|------|
| thumbnail-writer | 5개 앵글별 썸네일 헤드라인 | generate-thumbnail, evaluate-copy | `_workspace/01_thumbnails.json` |
| body-writer | 본문 3장 + CTA | generate-body, evaluate-copy | `_workspace/02_body.json` |
| image-renderer | 배경이미지 + 카드 5장 PNG | generate-image | `output/{folder}/` |

## 워크플로우

### Phase 0: 컨텍스트 확인

1. `_workspace/` 존재 여부 확인
2. 실행 모드 결정:
   - `_workspace/` 없음 → **초기 실행** (Phase 1로)
   - `_workspace/` 있음 + 부분 수정 요청 → **부분 재실행** (해당 에이전트만)
   - `_workspace/` 있음 + 새 입력 → **새 실행** (`_workspace/`를 `_workspace_{timestamp}/`로 이동)

### Phase 1: 준비

1. 사용자 입력 파싱:
   - **페르소나** (필수)
   - **욕구** (필수)
   - **인지단계** (선택, 기본값 problem-aware)
   - **--auto** (선택, 승인 건너뛰기)
2. `_workspace/00_input/params.json`에 저장:
   ```json
   { "persona": "...", "desire": "...", "awareness": "...", "auto_approve": false }
   ```

### Phase 2: 썸네일 생성

```
Agent(
  description: "썸네일 헤드라인 생성",
  subagent_type: "thumbnail-writer",
  model: "opus",
  prompt: "카드뉴스 썸네일 헤드라인을 생성하라.
    _workspace/00_input/params.json을 Read로 읽고,
    config.yaml에서 서비스 정보를 확인하라.
    generate-thumbnail 스킬과 evaluate-copy 스킬을 따라 수행하라.
    결과를 _workspace/01_thumbnails.json에 저장하고 구글 시트에 추가하라."
)
```

완료 후 사용자에게 5개 앵글 헤드라인 + 자기 평가 점수를 요약 표시.

### Phase 3: 썸네일 승인 대기

- **--auto**: 모든 행을 "썸네일 승인"으로 변경
- **수동**: "구글 시트 K열(썸네일 상태)을 '썸네일 승인'으로 변경 후 알려주세요" 안내. 사용자 확인 후 진행.

### Phase 4: 본문 생성

```
Agent(
  description: "본문 및 CTA 생성",
  subagent_type: "body-writer",
  model: "opus",
  prompt: "카드뉴스 본문을 생성하라.
    _workspace/01_thumbnails.json을 Read로 읽고 승인된 앵글을 확인하라.
    generate-body 스킬과 evaluate-copy 스킬을 따라 수행하라.
    결과를 _workspace/02_body.json에 저장하고 구글 시트를 업데이트하라."
)
```

### Phase 5: 본문 승인 대기

Phase 3과 동일한 방식.

### Phase 6: 이미지 생성 + 카드 렌더링

```
Agent(
  description: "배경이미지 생성 및 카드 렌더링",
  subagent_type: "image-renderer",
  model: "opus",
  prompt: "_workspace/01_thumbnails.json과 _workspace/02_body.json을 읽어
    generate-image 스킬을 따라 수행하라.
    Gemini API로 배경이미지를 생성하고,
    HTML+Playwright로 카드 5장을 output/{folder_name}/에 렌더링하라."
)
```

### Phase 7: 완료 보고

```
카드뉴스 생성이 완료되었습니다.

- 세트: {N}개
- 출력: output/{folder_name}/
  01_thumbnail.png ~ 05_cta.png
```

## 데이터 흐름

```
[사용자] → params.json
              │
              ▼
       thumbnail-writer → 01_thumbnails.json + 구글시트
              │
       [승인 대기]
              │
              ▼
         body-writer → 02_body.json + 구글시트
              │
       [승인 대기]
              │
              ▼
       image-renderer → 03_images/ + output/{folder}/
              │
              ▼
         [완료 보고]
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 에이전트 실패 | 에러 메시지와 함께 사용자에게 보고. 재시도 제안 |
| 시트 접속 불가 | JSON 파일 기반으로만 진행 |
| 승인 행 없음 | 사용자에게 시트 확인 요청 |
| 패키지 미설치 | 에이전트가 필요 패키지를 자동 설치 후 진행 |

## 테스트 시나리오

### 정상 흐름
1. "카드뉴스 만들어줘 --persona '20대 대학생' --desire 'AI 활용법' --auto"
2. 썸네일 5개 앵글 생성 (각 7점 이상) → 자동 승인
3. 본문+CTA 생성 (톤 일관성 7점 이상) → 자동 승인
4. Gemini 배경이미지 + 카드 5장 PNG
5. output/2026-04-14_대학생_공감/01~05.png 생성

### 에러 흐름
1. 썸네일 검증 3회 실패 → 최고 점수 버전(6.5점) 채택
2. "점수 6.5점으로 저장. 시트에서 확인 후 승인 여부 결정해주세요." 안내
