---
name: generate-image
description: "카드뉴스 배경이미지 생성 및 카드 PNG 렌더링. Gemini API로 배경이미지를 생성하고, HTML+Playwright로 카드 5장을 렌더링한다. 이미지 생성, 카드 렌더링, PNG 만들기 요청 시 사용."
---

# Generate Image — 배경이미지 생성 + 카드 렌더링

## 절차

### Step 1: 데이터 수집

`_workspace/01_thumbnails.json`과 `_workspace/02_body.json`을 Read로 읽는다.

### Step 2: 이미지 프롬프트 2단계 변환

직접 프롬프트를 변환한다 (별도 API 호출 불필요 — 에이전트 자신이 변환).

**2-1. 헤드라인 → 시각 장면**
한국어 헤드라인을 구체적인 영어 시각 묘사로 변환한다.
- 추상적이고 그래픽적인 표현을 선호
- 사람 얼굴 포함 금지
- 예: "마케터 필수 AI 툴" → "Three glowing keycaps with AI symbols on a sleek dark keyboard"

**2-2. 페르소나 → 스타일/색감**
타겟에 어울리는 시각 스타일과 색감을 영어로 묘사한다.
- 예: "2030 주니어 마케터" → "Modern high-tech aesthetic with cyan and purple neon accents"

### Step 3: Gemini API로 배경이미지 생성

`references/gemini-api-guide.md`를 참조하여 Bash로 인라인 Python을 실행한다:

```bash
python3 -c "
import google.generativeai as genai
import os
from pathlib import Path

genai.configure(api_key=os.environ.get('GEMINI_API_KEY') or open('.env').read().split('GEMINI_API_KEY=')[1].split('\n')[0])

model = genai.ImageGenerationModel('imagen-3.0-generate-002')
result = model.generate_images(
    prompt='장면묘사 + 스타일묘사 + no text, no watermark, no letters, card news background',
    number_of_images=1,
    aspect_ratio='1:1'
)

Path('_workspace/03_images').mkdir(parents=True, exist_ok=True)
result.images[0]._pil_image.save('_workspace/03_images/bg.png')
print('배경이미지 생성 완료')
"
```

Gemini 패키지가 없으면 먼저 `pip3 install google-generativeai`를 실행한다.

### Step 4: 카드 5장 HTML → PNG 렌더링

각 카드에 대해 HTML을 직접 작성하고 Playwright로 스크린샷을 찍는다. 별도 템플릿 파일 없이 인라인 HTML을 사용한다.

**카드별 렌더링:**
| 카드 | 배경 | 내용 |
|------|------|------|
| 01_thumbnail | Gemini 배경 | 헤드라인 3줄 + 칩 2개 |
| 02_body1 | Gemini 배경 | Card02 섹션타이틀 + 인풋텍스트 |
| 03_body2 | Gemini 배경 | Card03 섹션타이틀 + 인풋텍스트 |
| 04_body3 | Gemini 배경 | Card04 섹션타이틀 + 인풋텍스트 |
| 05_cta | 단색 그라데이션 | CTA 유도문구 + 팔로우 버튼 |

HTML 구조 핵심:
- 크기: 1080x1350px (config.yaml의 card.width, card.height)
- 배경: base64 인코딩된 이미지를 `background-image: url(data:image/png;base64,...)`로 삽입
- 오버레이: 반투명 검정 그라데이션으로 텍스트 가독성 확보
- 폰트: system-ui, 'Apple SD Gothic Neo', sans-serif

Playwright 렌더링:
```bash
python3 -c "
import asyncio
from playwright.async_api import async_playwright

async def render(html, path, w, h):
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page(viewport={'width': w, 'height': h})
        await page.set_content(html, wait_until='networkidle')
        await page.screenshot(path=path, full_page=False)
        await browser.close()

asyncio.run(render(html_content, output_path, 1080, 1350))
"
```

Playwright가 없으면 `pip3 install playwright && playwright install chromium`을 먼저 실행한다.

### Step 5: 결과 기록

출력 폴더 경로를 구글 시트 PNG 폴더 컬럼에 기록한다.

## 비용 참고

| 항목 | 모델 | 비용 |
|------|------|------|
| 배경이미지 | Gemini Imagen 3 | 무료 (Google AI Studio 기준) |
| 카드 렌더링 | Playwright (로컬) | 무료 |

상세 API 사용법은 `references/gemini-api-guide.md` 참조.
