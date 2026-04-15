---
name: generate-body
description: "카드뉴스 본문(Card02~04) + CTA(Card05) 생성. 승인된 썸네일 기반으로 본문을 작성하고 톤 일관성을 검증한다. 본문 작성, 본문 생성, CTA 만들기 요청 시 사용."
---

# Generate Body — 본문/CTA 생성

## 절차

### Step 1: 승인된 데이터 읽기

`_workspace/01_thumbnails.json`을 Read로 읽어 승인된 앵글의 헤드라인, 페르소나, 욕구를 파악한다.

### Step 2: 본문 생성

`references/prompt-template.md`의 구조를 참조하여 각 승인 행에 대해 본문을 생성한다.

**카드 구조:**
| 카드 | 역할 |
|------|------|
| Card02 | 문제 제기/공감 — 페르소나의 현재 상황 묘사 |
| Card03 | 해결책/인사이트 — 핵심 방법이나 새로운 관점 |
| Card04 | 구체적 방법/증거 — 실행 가능한 단계, 숫자, 사례 |
| Card05 | CTA — 팔로우/저장/공유 유도문구 |

### Step 3: 검증

evaluate-copy 스킬의 절차에 따라 규칙 검증 → 자기 평가(톤 일관성 중심)를 수행한다.

### Step 4: 저장

1. `_workspace/02_body.json`에 Write로 저장
2. Bash로 인라인 Python을 실행하여 구글 시트 해당 행의 Card02~05 컬럼을 업데이트

## 출력 규칙

- 섹션타이틀: 20자 이내
- 인풋텍스트: 150자 이내
- CTA 유도문구: 30자 이내
- 말투: 썸네일과 동일한 체 유지
