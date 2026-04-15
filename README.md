# Card News Agent — 카드뉴스 자동화 하네스

페르소나 + 욕구를 입력하면, AI가 카드뉴스 5장 세트를 자동으로 생성하는 Claude Code 하네스.

**별도 코드 없이 `.md` 파일만으로 구성.** 에이전트가 실행 시 필요한 코드를 직접 작성하고 실행한다.

## 전체 흐름

```
페르소나 + 욕구 입력
        │
        ▼
  thumbnail-writer        5개 앵글별 헤드라인 생성 + 자기 검증 루프
        │
  [구글시트 승인 대기]     ← Human-in-the-loop (--auto 로 건너뛰기)
        │
        ▼
    body-writer           본문 3장 + CTA 생성 + 톤 일관성 검증
        │
  [구글시트 승인 대기]
        │
        ▼
   image-renderer         Gemini 배경이미지 + 카드 5장 PNG 렌더링
        │
        ▼
   output/{날짜}_{페르소나}/
   ├── 01_thumbnail.png
   ├── 02_body1.png
   ├── 03_body2.png
   ├── 04_body3.png
   └── 05_cta.png
```

## 프로젝트 구조

```
card-news-agent/
│
├── CLAUDE.md                              ← 하네스 진입점 (트리거 규칙)
├── config.yaml                            ← 서비스 정보, 이미지 모델, 카드 크기
├── .env.example                           ← API 키 템플릿
│
├── .claude/
│   ├── agents/                            ← 에이전트 = "누가 하는가"
│   │   ├── thumbnail-writer.md
│   │   ├── body-writer.md
│   │   └── image-renderer.md
│   │
│   └── skills/                            ← 스킬 = "어떻게 하는가"
│       ├── cardnews-orchestrator/
│       │   └── SKILL.md                   전체 파이프라인 오케스트레이터
│       ├── generate-thumbnail/
│       │   ├── SKILL.md                   썸네일 헤드라인 생성 절차
│       │   └── references/prompt-template.md
│       ├── generate-body/
│       │   ├── SKILL.md                   본문/CTA 생성 절차
│       │   └── references/prompt-template.md
│       ├── evaluate-copy/
│       │   └── SKILL.md                   공유 자기 검증 루프
│       └── generate-image/
│           ├── SKILL.md                   Gemini 이미지 + 카드 렌더링
│           └── references/gemini-api-guide.md
│
├── credentials/                           ← 구글 서비스 계정 JSON (gitignore)
├── _workspace/                            ← 에이전트 간 중간 산출물 (gitignore)
├── output/                                ← 최종 카드 PNG (gitignore)
└── docs/                                  ← 참조 문서
```

**코드 파일이 없다.** scripts/, templates/, requirements.txt 없음.
에이전트가 실행 시 필요한 패키지를 설치하고, HTML/Python을 직접 작성하여 실행한다.

## Setup

### 1. 사전 요구사항

- Python 3.10+
- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)

### 2. API 키 설정

```bash
cp .env.example .env
# .env 파일을 열어 실제 키 입력
```

| 키 | 발급처 | 용도 |
|----|--------|------|
| `GEMINI_API_KEY` | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) | 배경이미지 생성 (Gemini Imagen 3) |

> Claude API 키는 별도 설정 불필요 — Claude Code가 자체 인증을 사용한다.

### 3. 구글 시트 설정

**3-1. 서비스 계정 생성**

1. [console.cloud.google.com](https://console.cloud.google.com) → 프로젝트 생성
2. API 및 서비스 → 라이브러리 → "Google Sheets API" 사용 설정
3. 사용자 인증 정보 → 서비스 계정 생성 → JSON 키 다운로드
4. JSON 파일을 `credentials/service-account.json`으로 이동

**3-2. 시트 생성**

1. 새 스프레드시트 생성, 시트 이름: `카드뉴스 결과물`
2. 1행 헤더 22개:

| A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|
| 날짜 | 폴더명 | 페르소나 | 욕구 | 인지단계 | 펀넬 | 앵글 |

| H | I | J | K |
|---|---|---|---|
| Card01 헤드라인 | Card01 칩1 | Card01 칩2 | 썸네일 상태 |

| L | M | N | O | P | Q |
|---|---|---|---|---|---|
| Card02 섹션타이틀 | Card02 인풋텍스트 | Card03 섹션타이틀 | Card03 인풋텍스트 | Card04 섹션타이틀 | Card04 인풋텍스트 |

| R | S | T | U | V |
|---|---|---|---|---|
| Card05 CTA 유도문구 | 컨셉 | 기대반응 | 본문 상태 | PNG 폴더 |

3. K열: 드롭다운 → `썸네일 대기, 썸네일 승인`
4. U열: 드롭다운 → `본문 대기, 본문 승인`
5. 서비스 계정 이메일에 편집자 권한으로 공유

**3-3. config.yaml 수정**

```yaml
google_sheet:
  url: "https://docs.google.com/spreadsheets/d/시트ID/edit"
```

### 4. 서비스 정보 설정

`config.yaml`을 본인 브랜드에 맞게 수정:

```yaml
service_name: "내 브랜드"
service_desc: "브랜드 설명"
brand_label: "브랜드 레이블"
```

### 5. 체크리스트

- [ ] Python 3.10+ (`python3 --version`)
- [ ] Claude Code CLI 설치
- [ ] `.env` 생성 + `GEMINI_API_KEY` 입력
- [ ] `credentials/service-account.json` 배치
- [ ] 구글 시트 생성 + 헤더 22개 + 드롭다운
- [ ] 구글 시트를 서비스 계정에 공유 (편집자)
- [ ] `config.yaml`에 시트 URL + 서비스 정보 입력

## 실행

```bash
claude    # 프로젝트 디렉토리에서 Claude Code 시작

# 자동 모드
> 카드뉴스 만들어줘 --persona "20대 대학생" --desire "AI를 더 잘 활용하고 싶다" --auto

# 수동 모드 (구글시트에서 승인 후 진행)
> 카드뉴스 만들어줘 --persona "3년차 마케터" --desire "업무 자동화를 시작하고 싶다"

# 부분 재실행
> 썸네일만 다시 생성해줘
> 이미지 재생성해줘
```

## 하네스 설계

### 코드 기반 vs 하네스 기반

| 관점 | 코드 기반 (jay.pm.ai) | 하네스 기반 (이 프로젝트) |
|------|----------------------|------------------------|
| 에이전트 정의 | Python 파일이 곧 에이전트 | `.md` 파일로 역할 정의 |
| 프롬프트 관리 | `prompts/` 폴더에 분리 | `skills/references/`에 통합 |
| 수정 방법 | 코드를 읽고 수정 | 마크다운만 편집 |
| 코드 관리 | scripts/ 폴더 유지 필요 | 에이전트가 실행 시 직접 생성 |
| 확장 | 새 .py 파일 작성 | 새 .md 파일 추가 |

### 자기 검증 루프 (evaluate-copy)

```
카피 생성 → 규칙 검증 → 자기 평가(10점) → 7점 이상? → 저장
   ↑          │ 실패        │ 7점 미만
   └────── 피드백 + 재생성 (최대 3회) ──┘
```

- **Feedforward**: 규칙을 프롬프트에 명시 (글자수, 줄수)
- **Feedback**: 자기 평가 후 피드백 반영 재작성
- thumbnail-writer와 body-writer가 **공유**하는 스킬

## 참조

- [카드뉴스 에이전트 가이드북](docs/) (jay.pm.ai)
- [상세페이지 현지화 자동화](https://zoey.day/blog?post=qrx6zk258k4k4mv314y5) (zoey.day)
