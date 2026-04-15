# 본문/CTA 생성 프롬프트 구조

## 프롬프트에 포함할 요소

1. **서비스 컨텍스트**: service_name, service_desc
2. **썸네일 정보**: 헤드라인 3줄, 앵글, 페르소나, 욕구, 칩태그
3. **카드 구조 안내**: Card02(공감) → Card03(인사이트) → Card04(방법) → Card05(CTA)

## 규칙

- 섹션타이틀 20자 이내
- 인풋텍스트 150자 이내
- CTA 유도문구 30자 이내
- 말투: 썸네일 헤드라인과 동일한 체 (~요체 또는 ~다체)
- 썸네일에서 약속한 내용을 본문에서 반드시 다룰 것

## 응답 JSON 구조

```json
{
  "card02": { "section_title": "제목", "input_text": "본문" },
  "card03": { "section_title": "제목", "input_text": "본문" },
  "card04": { "section_title": "제목", "input_text": "본문" },
  "card05_cta": "CTA 유도문구"
}
```
