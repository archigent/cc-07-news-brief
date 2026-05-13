# 영상에서 사용한 프롬프트 5개 (Claude Code/Codex 한 마디 셋업)

영상 시연에서 사용한 프롬프트 원문. 카테고리는 본인 분야로 자유 변경.

본 skill 기본 구성 = **Discord 매일 알림 + Google Sheets 매일 누적 통합** (영상 시연 실 운영 시스템).

---

## Step 1 — RSS 수집기 빌드

```
본인 관심 분야 한국 뉴스 6개 카테고리 — 정치, 사회, 경제, 기술, 연예, 문화 — 각각의 RSS 주소를 찾아서 최근 24시간 안에 올라온 기사들을 한 번에 수집하는 파이썬 스크립트 만들어줘. 결과는 JSON 파일로 저장. feedparser 사용. 사이트 한두 개 응답 안 해도 나머지는 정상 수집되도록 안전장치 포함.
```

**카테고리 변경 예시**:
- 투자: `주식, 부동산, 코인, 매크로, 글로벌, 정책`
- 연예: `K팝, 드라마, 영화, 셀럽, 예능, 해외엔터`
- 비즈니스: `AI, 스타트업, 마케팅, 매출, 트렌드, 리더십`

---

## Step 2 — LLM 요약·분류

```
방금 수집한 기사들을 LLM API로 보내서 한국어 3줄로 요약하고 6개 카테고리 중 하나로 분류해줘. 한국어 3줄 = 사실 한 줄, 맥락 한 줄, 시사점 한 줄 구조. 결과는 원본 JSON에 summary_ko랑 category 필드 추가해서 저장. 키는 .env에서 LLM_API_KEY로 읽어와. 짧은 시간에 너무 많이 요청 보내면 차단되니까 요청 사이에 0.5초 sleep 넣어줘. 진행률 표시도 같이.
```

---

## Step 3 — Discord 알림

```
분류된 결과를 디스코드 웹후크로 보내. 카테고리별 섹션 나누고 각 기사는 제목·한국어 3줄 요약·원문 출처 링크·출처 매체명 같이 보내. 분야마다 색상 다르게 (정치 빨강, 사회 주황, 경제 초록, 기술 파랑, 연예 보라, 문화 청록). webhook URL은 .env의 DISCORD_WEBHOOK_URL에서 읽어와. embed는 한 메시지당 최대 10개니까 그 이상이면 메시지 분할.
```

---

## Step 4 — Google Sheets 누적

```
방금 분류된 결과를 Google Sheets에도 매일 누적해줘. gspread + google-auth 사용. credentials.json은 .env의 GOOGLE_CREDENTIALS_PATH에서, 시트 ID는 SPREADSHEET_ID에서 읽어와. 시트 두 개 자동 생성·관리:
- "사용된뉴스": 발행일·카테고리·제목·한국어 3줄 요약·원문 링크·출처 매체 6컬럼, 매일 행 추가
- "브리핑히스토리": 발송시각·기사건수·상태 3컬럼, 매일 1행 추가
시트가 없으면 자동 생성. Service Account 이메일이 시트에 편집자로 공유 안 되어 있으면 권한 에러 친절한 메시지 출력.
```

---

## Step 5 — GitHub Actions 자동 실행

```
이 코드를 매일 아침 8시 한국 시각에 자동 실행하는 깃허브 액션 만들어줘. 키 4개는 시크릿(LLM_API_KEY, DISCORD_WEBHOOK_URL, GOOGLE_CREDENTIALS, SPREADSHEET_ID)에서 가져오게. GOOGLE_CREDENTIALS는 credentials.json 파일 내용 JSON 그대로 들어오니까 코드에서 json.loads로 파싱. UTC 기준이니까 한국 8시 = UTC 23시(전날)로 적어줘. 수동 실행도 가능하게 workflow_dispatch도 같이. pip install에 feedparser·requests·gspread·google-auth 포함.
```

---

## 마지막 — 본인 분야 매핑 한 마디

위 5 프롬프트를 한 번에 SKILL.md 기반으로 처리하고 싶으면 Claude Code/Codex에 이 한 마디만:

```
이 skill로 [본인 분야] 셋업해줘. 카테고리는 [카테고리 6개 콤마 구분]로.
LLM 키 [본인 키], Discord webhook [본인 URL], credentials.json은 [본인 경로], 시트 ID는 [본인 ID].
```

예시:
- `이 skill로 투자 분야 셋업해줘. 카테고리는 주식·부동산·코인·매크로·글로벌·정책. LLM 키 openrouter_xxx, Discord webhook https://discord.com/api/webhooks/yyy, credentials.json은 ./credentials.json, 시트 ID는 1QCuC...N1kTXM778elNlhE.`

---

## Step 6 — 매일 사용 흐름 (Claude Code 토론 사이클)

시트 누적 시작 후 매일·주간 토론 활용:

```
오늘 시트 가져와서 본인 분야 핵심 짚어줘
```

```
이번 주 시트에서 트렌드 3가지 뽑아줘
```

```
이번 달 [카테고리] 누적 흐름 분석해줘
```

매일 자료 누적 = 본인 영역 인사이트 자산.
