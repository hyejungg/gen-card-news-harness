# Gemini 이미지 생성 API 가이드

## 인증

API 키를 `.env`의 `GEMINI_API_KEY`에 저장한다.
발급: https://aistudio.google.com/apikey

```python
import google.generativeai as genai
genai.configure(api_key="YOUR_API_KEY")
```

## 이미지 생성 (Imagen 3)

```python
from google import genai
from google.genai import types

client = genai.Client(api_key="YOUR_API_KEY")

response = client.models.generate_images(
    model="imagen-3.0-generate-002",
    prompt="your prompt here",
    config=types.GenerateImagesConfig(
        number_of_images=1,
        aspect_ratio="1:1",
    )
)

# PIL Image로 저장
response.generated_images[0].image._pil_image.save("output.png")
```

## 텍스트 포함 이미지 생성 (Gemini 2.0 Flash)

Gemini 모델 자체로도 이미지를 생성할 수 있다. 텍스트가 포함된 이미지가 필요할 때 유용하다.

```python
from google import genai

client = genai.Client(api_key="YOUR_API_KEY")

response = client.models.generate_content(
    model="gemini-2.0-flash-exp",
    contents="Generate an image of...",
    config=types.GenerateContentConfig(
        response_modalities=["IMAGE"],
    )
)

# 이미지 저장
for part in response.candidates[0].content.parts:
    if part.inline_data:
        with open("output.png", "wb") as f:
            f.write(part.inline_data.data)
```

## 프롬프트 작성 팁

1. **영어로 작성**: 영어 프롬프트가 품질이 높다
2. **구체적 시각 묘사**: "beautiful" 대신 "soft gradient with floating geometric shapes, volumetric lighting"
3. **No text 명시**: 끝에 `no text, no watermark, no letters` 추가
4. **스타일 키워드**: `minimalist`, `flat design`, `3D render`, `glassmorphism`

## 에러 처리

- `429 RESOURCE_EXHAUSTED`: 30초 대기 후 재시도
- `400 INVALID_ARGUMENT`: 프롬프트에 금지 콘텐츠 확인 (사람 얼굴 등)
- 이미지 생성 실패 시: PIL로 그라데이션 배경 대체

## 패키지 설치

```bash
pip3 install google-generativeai Pillow
```
