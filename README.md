# 매일 아침 뉴스 자동 브리핑 — News Brief Skill

매일 본인 관심 분야 뉴스 6개 카테고리를 RSS로 수집 → LLM으로 한국어 3줄 요약 + 분야 분류 → **Discord 매일 알림 + Google Sheets 매일 누적** 동시 발송. 매일 정해진 시각 GitHub Actions가 자동 실행. 영상에서 시연된 실 운영 시스템 그대로.

영상: https://youtu.be/Qlrtcw6YNzc

---

> ⚠️ **이 Skill의 사용 방식**
>
> 이 Skill은 **그대로 실행하는 자판기가 아니라, 본인 환경에 맞게 fork·개조하는 레퍼런스**입니다.
> 본인의 관심 분야·언어·요약 톤·필터 룰에 맞춰 SKILL.md 본문과 prompts.md를 손봐서 써주세요.
> 처음에는 그대로 한 번 돌려보는 것도 OK — 진짜 가치는 본인 워크플로우에 흡수해서 매일 본인이 쓰는 시스템이 됐을 때 나옵니다.

---

## 다운로드 + 설치

1. 이 페이지 상단의 **Code → Download ZIP** 클릭
2. 압축 풀기
3. 풀린 폴더를 `~/.claude/skills/news-brief/`로 복사 (Claude Code 사용자)
   또는 `~/.codex/skills/news-brief/`로 복사 (Codex 사용자)

---

## 사용법

### 한 줄 셋업 (가장 빠른 방법)

Claude Code 또는 Codex 실행하고 한 마디만:

```
이 skill로 [본인 분야] 셋업해줘. 카테고리는 [카테고리 6개]로.
```

**예시**:
- `이 skill로 투자 분야 셋업해줘. 카테고리는 주식·부동산·코인·매크로·글로벌·정책로.`
- `이 skill로 연예 분야로. K팝·드라마·영화·셀럽·예능·해외엔터.`
- `이 skill로 비즈니스 분야로. AI·스타트업·마케팅·매출·트렌드·리더십.`

Claude/Codex가 본 skill을 읽고 본인 분야 RSS 주소·LLM 호출·Discord webhook·GitHub Actions YAML까지 자동 생성합니다.

### 단계별 빌드 (영상 따라하기)

영상 순서대로 5 프롬프트를 하나씩 던지면 됩니다. 프롬프트 원문은 `templates/prompts.md` 참조.

---

## 사전 발급 키 4개 (총 6분)

### 1. LLM API 키 — 무료 우선

**OpenRouter (무료, 가장 간단)**:
1. https://openrouter.ai/ → Sign in (Google 계정 OK)
2. https://openrouter.ai/keys → Create Key
3. 키 복사

**대안**:
- Gemini 무료: https://aistudio.google.com/apikey
- Claude 유료: https://console.anthropic.com/
- GPT 유료: https://platform.openai.com/api-keys

매일 60개 기사 정도면 무료 한도 안에서 충분합니다.

### 2. Discord webhook 주소

1. 본인 디스코드 서버 만들기 (없다면)
2. 알림 받을 채널 우클릭 → **Edit Channel** → **Integrations** → **Webhooks** → **New Webhook**
3. **Copy Webhook URL**

### 3. Google Service Account credentials.json

1. https://console.cloud.google.com → 신규 프로젝트 (1분)
2. "API 및 서비스" → Google Sheets API 사용 설정
3. "사용자 인증 정보 만들기" → 서비스 계정 → 만들기
4. 서비스 계정 클릭 → "키" 탭 → "키 추가" → JSON → 다운로드 → `credentials.json` 저장

### 4. Google Sheets ID + 권한 공유

1. https://sheets.google.com → 새 시트 생성
2. URL `docs.google.com/spreadsheets/d/[이 부분이 ID]/edit` 에서 ID 복사
3. 시트 우상단 "공유" → credentials.json의 `client_email` (예: `your-bot@your-project.iam.gserviceaccount.com`) → **편집자 권한** 부여 (없으면 권한 에러)

### 5. GitHub 저장소 + Secrets 4개

1. 본인 GitHub 새 저장소 만들기 (Public 또는 Private 둘 다 OK)
2. 위에서 다운받은 skill 파일들을 그 저장소에 푸시
3. 저장소 **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
4. 등록 4건:
   - 이름: `LLM_API_KEY`, 값: 1번에서 받은 키
   - 이름: `DISCORD_WEBHOOK_URL`, 값: 2번에서 받은 webhook URL
   - 이름: `GOOGLE_CREDENTIALS`, 값: 3번 credentials.json **파일 내용 JSON 그대로** 복사·붙여넣기
   - 이름: `SPREADSHEET_ID`, 값: 4번에서 복사한 시트 ID

---

## 키 보관 (.env)

로컬 테스트할 때는 `.env` 파일을 만들어 키를 보관합니다.

1. `templates/.env.example`를 같은 폴더에 `.env`로 복사
2. `YOUR_LLM_API_KEY_HERE` 자리를 본인 키로 교체
3. `.gitignore`에 `.env` 추가 (이미 포함돼 있음). 깃허브에 .env 파일 올라가면 키 노출 위험

---

## 매일 자동 실행

GitHub Actions가 알아서 매일 정해진 시각 실행합니다.

- 기본: 매일 한국 시각 **오전 8시** (UTC 23시)
- 시간 변경 = `.github/workflows/daily.yml` 안 cron 표기만 수정
- 수동 실행 = 저장소 **Actions** 탭 → workflow 선택 → **Run workflow** 클릭

본인 PC가 꺼져 있어도 깃허브 서버가 실행해줍니다.

---

## 막힌 부분

영상 댓글에 어디서 멈췄는지 적어주세요. 답변 드리겠습니다.

또는 카톡방: https://archigent.co/chat

---

## 매일 사용 흐름 (Discord 알림 + Sheets 토론 사이클)

1. **매일 8시 Discord 알림** 도착 (카테고리별 색상)
2. **출근길 폰으로 5분 훑기** — 본인 분야만 추리고 다른 분야 스킵
3. **사무실 도착 → Claude Code 토론** — 한 마디:
   - `오늘 시트 가져와서 본인 분야 핵심 짚어줘`
   - `이번 주 시트에서 트렌드 3가지 뽑아줘`
   - `이번 달 [카테고리] 누적 흐름 분석해줘`
4. **주말 회고** — 시트 누적 데이터로 주간·월간 트렌드 직접 분석

매일 자료 누적 = 본인 영역 인사이트 자산.

> **왜 MCP가 아니라 Service Account?** Google Sheets MCP 서버도 있지만, MCP는 Claude Code를 켜고 대화하는 중에만 도구를 쓸 수 있어요. 본 skill은 본인 PC가 꺼져 있어도 매일 8시 GitHub Actions가 자동으로 시트를 채우는 게 본질이라, Claude Code 의존도 0인 Service Account 방식으로 갑니다. 한 번 셋업해두면 매일 자동 누적 + Claude Code 토론 둘 다 가능.

---

## 본인에게 맞게 개조하기 (fork 가이드)

처음 한 번은 그대로 돌려도 되지만, 진짜 가치는 본인 환경에 맞춘 다음에 나옵니다. 4자리 손보시면 됩니다:

- **`SKILL.md` 본문 카테고리 6개** → 본인 관심 분야로 교체 (예: 정치·사회·경제·기술·연예·문화 → 주식·부동산·코인·매크로·글로벌·정책)
- **`templates/prompts.md` 요약 톤·길이** → 3줄 사실·맥락·시사점이 무거우면 1줄 핵심만, 더 깊게 원하면 5줄까지 재작성
- **`.env`** → 본인 LLM 키·Discord webhook·credentials.json 경로·시트 ID로 교체 (`.env.example`을 `.env`로 복사 후 채움)
- **수신 시간·요일** → `templates/github-actions-daily.yml` cron 한 줄 본인 일과에 맞춰 변경

본인 합격 기준(어떤 알림이 합격이고 어떤 게 노이즈인지)도 SKILL.md 트러블슈팅 섹션에 본인 룰로 다시 적어두면, 다음 번 Claude/Codex 호출 때 본인 기준이 재현됩니다.

---

## 참고

- 영상 자료 모음 (영구 인덱스): https://archigent.co/prompts
- 이 skill의 상세 패턴 코드·트러블슈팅: `SKILL.md` 참조
- 영상에서 사용한 프롬프트 원문: `templates/prompts.md` 참조
- 자동 실행 워크플로우 템플릿: `templates/github-actions-daily.yml`
4. 주말 → "이번 주 시트에서 트렌드 3가지 뽑아줘" → 주간 회고
5. 매일 자료 누적 → 본인 영역 인사이트 자산
