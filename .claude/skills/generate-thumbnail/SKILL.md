---
name: generate-thumbnail
description: "카드뉴스 썸네일 헤드라인 생성. 페르소나+욕구+인지단계로 5개 마케팅 앵글별 헤드라인 3줄 + 칩태그 2개를 생성한다. 썸네일 생성, 헤드라인 만들기, 카드뉴스 카피 요청 시 사용."
---

# Generate Thumbnail — 썸네일 헤드라인 생성

## 절차

### Step 1: 입력 수집

`_workspace/00_input/params.json`을 Read로 읽어 페르소나, 욕구, 인지단계를 파악한다. config.yaml에서 service_name, service_desc, brand_label도 읽는다.

### Step 2: 5개 앵글별 헤드라인 생성

`references/prompt-template.md`의 프롬프트 구조를 참조하여 한 번에 5개 앵글을 생성한다.

**5개 마케팅 앵글:**
| 앵글 | 전략 |
|------|------|
| 공감 | 페르소나의 현재 상황에 공감 |
| 공포 | 안 하면 뒤처진다는 위기감 |
| 이익 | 구체적 혜택/숫자 제시 |
| 편의 | 쉽고 간편함 강조 |
| 사회증거 | 다른 사람도 하고 있다 |

### Step 3: 검증

evaluate-copy 스킬의 절차에 따라 1단계 규칙 검증 → 2단계 자기 평가를 수행한다.

### Step 4: 저장

1. `_workspace/01_thumbnails.json`에 Write로 저장
2. Bash로 인라인 Python을 실행하여 구글 시트에 5행 추가:

```bash
python3 -c "
import gspread, json, os
from google.oauth2.service_account import Credentials
from datetime import date

creds = Credentials.from_service_account_file(
    'credentials/service-account.json',
    scopes=['https://www.googleapis.com/auth/spreadsheets']
)
gc = gspread.authorize(creds)
sh = gc.open_by_url('시트URL')
ws = sh.worksheet('카드뉴스 결과물')

data = json.load(open('_workspace/01_thumbnails.json'))
for angle in data['angles']:
    ws.append_row([
        str(date.today()), data['folder_name'], data['persona'], data['desire'],
        data['awareness'], '', angle['angle'],
        '\n'.join(angle['headline']), angle['chip1'], angle['chip2'],
        '썸네일 대기', '', '', '', '', '', '', '', '', '', '', ''
    ], value_input_option='USER_ENTERED')
print('시트 저장 완료')
"
```

config.yaml의 google_sheet.url을 시트URL 자리에 넣는다.

## 출력 규칙

- 헤드라인: 정확히 3줄, 각 줄 한국어 7~12자
- 칩1: 페르소나 키워드 (#으로 시작)
- 칩2: 콘텐츠 키워드 (#으로 시작)
- 폴더명: `{YYYY-MM-DD}_{페르소나축약}_{앵글}`
