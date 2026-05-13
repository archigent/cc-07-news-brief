---
name: news-brief
description: 매일 아침 본인 관심 분야 뉴스를 RSS로 수집하고 LLM으로 한국어 3줄 요약·분야 분류한 다음 Discord 매일 알림 + Google Sheets 매일 누적. GitHub Actions로 매일 정해진 시각 자동 실행.
---

# News Brief Skill

매일 본인 관심 분야 6개 카테고리 뉴스를 자동으로 수집·요약·분류해서 **Discord 매일 알림 + Google Sheets 매일 누적** 두 매체 동시 발송하는 시스템 skill. 영상에서 시연된 실 운영 시스템 그대로.

> **이 Skill의 사용 방식**: 그대로 실행하는 자판기 ❌ / 본인 환경에 맞춰 **fork·개조하는 레퍼런스** ✅. 카테고리·요약 톤·필터 룰을 본인 분야로 교체한 다음 본인 운영 시스템으로 흡수해주세요. 처음 한 번은 그대로 돌려보는 것도 OK.

## 시스템 5단계

1. **RSS 수집** — 본인 관심 분야 6개 카테고리 사이트의 RSS를 자동으로 찾아서 최근 24시간 기사를 한 번에 수집
2. **LLM 요약·분류** — 각 기사를 한국어 3줄(사실·맥락·시사점)로 줄이고 6개 카테고리 중 하나로 다시 분류
3. **Discord 알림** — 분류된 결과를 카테고리별 색상으로 Discord 채널에 자동 게시 (출근길 폰 5분 훑기용)
4. **Google Sheets 누적** — 전체 기사 + 본문 + 한국어 3줄 요약을 매일 시트에 행 추가 (누적 자산 + Claude Code 토론용)
5. **GitHub Actions 자동 실행** — 매일 정해진 시각(예: 한국 8시)에 깃허브 서버가 위 흐름을 자동 실행

## 빌드 방법

사용자가 Claude Code 또는 Codex에 한 마디만 던지면 됩니다:

> "이 skill로 [본인 분야] 카테고리 6개로 셋업해줘. 카테고리는 [예: 주식·부동산·코인·매크로·글로벌·정책]."

Claude/Codex가 본 SKILL.md를 읽고 사용자 분야에 맞춰 자동으로:
- RSS 주소 검색 → 코드 추가
- LLM API 호출 코드 작성 (3줄 요약 + 카테고리 분류 프롬프트 포함)
- Discord webhook 전송 코드 작성
- Google Sheets 행 추가 코드 작성 (`gspread`)
- GitHub Actions YAML 생성 (시크릿 4개)

## 핵심 패턴

### Step 1 — RSS 수집 (Python)

```python
import feedparser
from datetime import datetime, timedelta, timezone

CATEGORIES = {
    # 사용자 분야에 맞게 변경
    "정치": ["https://www.yonhapnewstv.co.kr/category/news/politics/feed/", ...],
    "경제": ["https://www.mk.co.kr/rss/30100041/", ...],
    # ...
}

def collect_recent(hours=24):
    cutoff = datetime.now(timezone.utc) - timedelta(hours=hours)
    items = []
    for cat, urls in CATEGORIES.items():
        for url in urls:
            try:
                feed = feedparser.parse(url)
                for e in feed.entries:
                    pub = datetime(*e.published_parsed[:6], tzinfo=timezone.utc)
                    if pub >= cutoff:
                        items.append({
                            "title": e.title, "link": e.link,
                            "summary": e.get("summary", ""),
                            "source_category": cat,
                            "source": feed.feed.get("title", ""),
                            "published": pub.isoformat(),
                        })
            except Exception:
                continue  # 사이트 하나 다운돼도 나머지 정상 수집
    return items
```

**주의사항**:
- `feedparser` 설치 필요 (`pip install feedparser`)
- 사이트 응답 실패 시 건너뛰는 안전장치 (한 사이트 다운이 전체 실패 유발하면 안 됨)
- 24시간 필터로 중복 누적 방지

### Step 2 — LLM 요약·분류

```python
import os, time, json
import requests

LLM_API_KEY = os.environ["LLM_API_KEY"]
LLM_ENDPOINT = "https://openrouter.ai/api/v1/chat/completions"  # 또는 Claude·OpenAI

PROMPT = """
다음 기사를 한국어 3줄로 요약하고 6개 카테고리 중 하나로 분류해줘.

요약 형식:
- 사실 한 줄: 무슨 일이 있었나
- 맥락 한 줄: 왜 일어났나
- 시사점 한 줄: 그래서 어떤 의미인가

카테고리: 정치, 사회, 경제, 기술, 연예, 문화
(사용자가 지정한 분야로 교체)

출력 형식 (JSON):
{"summary": "사실\\n맥락\\n시사점", "category": "정치"}

기사 본문:
{title}
{summary}
"""

def summarize(items, batch_pause=0.5):
    out = []
    for i, it in enumerate(items):
        msg = PROMPT.format(title=it["title"], summary=it["summary"][:1500])
        r = requests.post(LLM_ENDPOINT, json={
            "model": "openai/gpt-4o-mini",  # 무료 또는 저비용 모델
            "messages": [{"role": "user", "content": msg}],
            "response_format": {"type": "json_object"},
        }, headers={"Authorization": f"Bearer {LLM_API_KEY}"}, timeout=30)
        result = r.json()["choices"][0]["message"]["content"]
        parsed = json.loads(result)
        it["summary_ko"] = parsed["summary"]
        it["category"] = parsed["category"]
        out.append(it)
        time.sleep(batch_pause)  # rate limit 회피
        print(f"  [{i+1}/{len(items)}] {it['title'][:40]}")
    return out
```

**주의사항**:
- 키는 반드시 `.env` 또는 환경변수로. 코드에 직접 넣지 말 것 (깃허브 노출 위험)
- `time.sleep(0.5)` rate limit 회피용. LLM 회사가 짧은 시간 안에 너무 많은 요청 받으면 일시 차단
- 무료 시작 권장: OpenRouter free tier 또는 Gemini free key (매일 70개 기사면 충분)

### Step 3 — Discord 알림

```python
DISCORD_WEBHOOK = os.environ["DISCORD_WEBHOOK_URL"]

COLOR = {
    "정치": 0xDC2626, "사회": 0xEA580C, "경제": 0x16A34A,
    "기술": 0x2563EB, "연예": 0x9333EA, "문화": 0x0891B2,
}

def post_to_discord(items):
    by_cat = {}
    for it in items:
        by_cat.setdefault(it["category"], []).append(it)
    for cat, arr in by_cat.items():
        embeds = [{
            "title": it["title"][:256],
            "description": it["summary_ko"][:2048] + f"\n\n출처: {it['source']}",
            "url": it["link"],
            "color": COLOR.get(cat, 0x6B7280),
        } for it in arr[:10]]  # Discord embed 한 메시지당 최대 10
        requests.post(DISCORD_WEBHOOK, json={
            "content": f"**{cat}** ({len(arr)}건)",
            "embeds": embeds,
        }, timeout=15)
```

**주의사항**:
- Discord webhook URL도 키처럼 `.env`에 보관 (노출되면 외부인이 본인 채널에 임의 메시지 전송 가능)
- Embed는 한 메시지당 최대 10개. 10개 넘으면 메시지 분할
- 카테고리별 색상은 사용자 분야로 자유 변경

### Step 4 — Google Sheets 누적

```python
import os, json
import gspread
from google.oauth2.service_account import Credentials

SCOPES = ['https://www.googleapis.com/auth/spreadsheets']
SPREADSHEET_ID = os.environ["SPREADSHEET_ID"]

def get_sheet():
    creds_json = os.environ["GOOGLE_CREDENTIALS"]
    creds = Credentials.from_service_account_info(json.loads(creds_json), scopes=SCOPES)
    gc = gspread.authorize(creds)
    return gc.open_by_key(SPREADSHEET_ID)

def append_to_sheets(items):
    sh = get_sheet()
    # 시트 1: 사용된뉴스 (전체 기사 누적, 중복 방지)
    used = sh.worksheet("사용된뉴스") if "사용된뉴스" in [w.title for w in sh.worksheets()] else sh.add_worksheet("사용된뉴스", 1, 8)
    rows = [[it["published"], it["category"], it["title"], it["summary_ko"], it["link"], it["source"]] for it in items]
    if rows:
        used.append_rows(rows, value_input_option="USER_ENTERED")
    # 시트 2: 브리핑히스토리 (발송 기록)
    history = sh.worksheet("브리핑히스토리") if "브리핑히스토리" in [w.title for w in sh.worksheets()] else sh.add_worksheet("브리핑히스토리", 1, 4)
    from datetime import datetime
    history.append_row([datetime.now().isoformat(), len(items), "Discord+Sheets 발송 완료"], value_input_option="USER_ENTERED")
    print(f"  Sheets 누적 완료: {len(items)}건")
```

**주의사항**:
- `GOOGLE_CREDENTIALS` 환경변수는 credentials.json **파일 내용 전체**를 JSON 문자열로 (GitHub Secrets에 그대로 붙여넣기)
- Service Account 이메일을 시트 "공유" → 편집자 권한 부여 의무. 안 하면 권한 에러
- 첫 실행 시 `사용된뉴스` / `브리핑히스토리` 시트 자동 생성
- Claude Code 토론용: 사용자가 "오늘 시트 가져와서 본인 분야 핵심 짚어줘" 명령 시 Claude가 직접 `gspread`나 다른 방법으로 시트 read

### Step 5 — GitHub Actions 자동 실행

`.github/workflows/daily.yml` (`templates/github-actions-daily.yml` 참조):

```yaml
name: Daily News Brief
on:
  schedule:
    - cron: '0 23 * * *'  # UTC 23시 = 한국 8시 (다음날)
  workflow_dispatch:      # 수동 실행 가능

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install feedparser requests gspread google-auth
      - env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          DISCORD_WEBHOOK_URL: ${{ secrets.DISCORD_WEBHOOK_URL }}
          GOOGLE_CREDENTIALS: ${{ secrets.GOOGLE_CREDENTIALS }}
          SPREADSHEET_ID: ${{ secrets.SPREADSHEET_ID }}
        run: python main.py
```

**주의사항**:
- GitHub Actions cron 시간 = **UTC 기준**. 한국 시각에 9시간 빼서 적기
- 키 4개 모두 `Repository Settings → Secrets and variables → Actions`에 등록. 코드에는 시크릿 이름만 참조
- 공개 저장소는 무제한 무료. 비공개도 월 2,000분 무료 (매일 1분 실행이면 평생 무료)

## 사용자 커스터마이징 1줄 프롬프트

Claude/Codex에 던질 한 마디 예시:

- "이 skill로 투자 분야 셋업해줘. 카테고리는 주식·부동산·코인·매크로·글로벌·정책로."
- "이 skill로 연예 분야로. K팝·드라마·영화·셀럽·예능·해외엔터."
- "이 skill로 비즈니스 분야로. AI·스타트업·마케팅·매출·트렌드·리더십."

Claude/Codex가 SKILL.md를 읽고 본 패턴 코드를 사용자 분야 RSS·카테고리·색상으로 자동 변환.

## 트러블슈팅

| 증상 | 원인·해결 |
|---|---|
| 기사 0건 수집 | RSS 주소 잘못. Claude에 "이 사이트 RSS 다시 찾아줘" |
| LLM 응답 401 | API 키 잘못 또는 만료. `.env` 확인 |
| LLM rate limit (429) | `time.sleep(0.5)` 값을 1.0이나 2.0으로 증가 |
| Discord 메시지 안 옴 | webhook URL 잘못 또는 채널 권한 |
| Sheets 권한 에러 | Service Account 이메일을 시트 "공유" 편집자에 추가 안 함. credentials.json `client_email` 확인 후 시트 공유 |
| Sheets 행 추가 안 됨 | `GOOGLE_CREDENTIALS` JSON 문자열 형식 오류. GitHub Secrets에 파일 내용 그대로 붙여넣었는지 확인 |
| GitHub Actions 실행 안 됨 | cron 시간 UTC인지 확인. 저장소 Actions 탭에서 수동 실행 테스트 |
| 같은 기사 반복 알림 | 24시간 필터 작동 중인지 확인. 또는 마지막 실행 시각 저장해 그 이후 기사만 처리 |

## 사전 발급 키 4개 (총 6분)

본 skill 기본 구성 = **Discord 매일 알림 + Google Sheets 매일 누적 통합**. 영상에서 보여드린 실 운영 시스템 그대로.

1. **LLM API 키** — OpenRouter(무료) / Gemini(무료) / Claude(유료) / GPT(유료) 중 1개
2. **Discord webhook 주소** — 본인 디스코드 서버 → 채널 설정 → 통합 → 웹후크
3. **Google Service Account credentials.json** — Google Cloud 콘솔 Service Account 생성 → JSON 키 다운로드 → Google Sheets API 권한
4. **Google Sheets ID** — 새 시트 생성 → URL에서 ID 추출 → Service Account 이메일에 편집자 권한 공유

5번째 = **GitHub 저장소** — 빈 저장소 생성 + Secrets 4개 등록 (LLM_API_KEY / DISCORD_WEBHOOK_URL / GOOGLE_CREDENTIALS / SPREADSHEET_ID)

발급 절차 상세 = `README.md` 참조.

---

## Sheets 통합 구성 본질

**왜 Discord + Sheets 둘 다인가**:
- **Discord = 매일 빠른 알림** (출근길 폰으로 본인 분야 5분 훑기용)
- **Google Sheets = 누적 자산** (전체 기사 + 본문 + 한국어 3줄 요약, 매일 행 추가)
- 두 매체 분리 = 휘발 알림 + 영구 자산 분리

**Sheets 시트 구조** (자동 생성):
- `대시보드` — 일일 종합 요약
- `사용된뉴스` — 중복 방지용 URL 기록
- `브리핑히스토리` — 발송 기록

**사용자 권장 흐름**:
1. 매일 8시 Discord 알림 도착
2. 출근길 폰으로 본인 분야 5분 훑기
3. 사무실 도착 → Claude Code 열기 → "오늘 시트 가져와서 본인 분야 핵심 짚어줘" → 본문 토론
4. 주말 → "이번 주 시트에서 트렌드 3가지 뽑아줘" → 주간 회고

### 시트 연결 — Google Service Account 방식 (고정)

매일 8시 GitHub Actions가 자동으로 시트에 행 추가. 사용자 PC 꺼져 있어도 작동.

**라이브러리**: `gspread` + `google-auth` (templates/github-actions-daily.yml에 기본 포함).

> **왜 MCP가 아니라 Service Account인가**: Google Sheets MCP도 있지만, MCP는 Claude Code를 켜고 대화하는 중에만 도구를 쓸 수 있음. 본 skill은 사용자 PC가 꺼져 있어도 매일 8시 GitHub Actions 서버가 자동으로 시트를 채우는 게 본질 — Claude Code 의존도 0. 그래서 OAuth 자격증명을 파일(credentials.json)로 떨어뜨려 서버가 직접 인증하는 Service Account 방식 채택.

### 한 마디 셋업 프롬프트

```
이 skill로 [본인 분야] 셋업해줘. 카테고리는 [6개].
LLM 키 [본인 키], Discord webhook [본인 URL], credentials.json은 [본인 경로], 시트 ID는 [본인 ID].
```

Claude/Codex가 RSS·LLM·Discord·Sheets·GitHub Actions 5요소를 한 번에 자동 셋업.
