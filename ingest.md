---
description: Brain Vault에 source 1건 ingest — raw 보존 + 요약 + wiki append + index/log 갱신. PDF/URL/Notion/마크다운 자동 감지.
argument-hint: <파일경로 | URL | 또는 직접 텍스트 붙여넣기>
allowed-tools: Read, Write, Edit, Bash, Agent, Glob, Grep, mcp__claude_ai_Notion__notion-fetch, WebFetch
---

# /ingest — Brain Vault 표준 ingest 워크플로우

사용자가 호출했으면 **즉시 시작**. 확인 질문 최소화(자율 진행 선호 — `feedback_autonomy.md`). 다만 vault 구조에 영향 큰 결정(신규 폴더 신설, 대용량 raw 복사, 다중 페이지 양산)은 한 번에 정리해서 짧게 확인.

## STEP 0 — 환경·입력 파악

1. **OS 감지**:
   ```bash
   uname -s 2>/dev/null || echo "Windows"
   ```
   - `Darwin` → macOS, vault: `/Users/aerok86/Library/Mobile Documents/iCloud~md~obsidian/Documents/Brain`
   - `Linux`/`MINGW`/`MSYS`/`CYGWIN` 또는 fallback → Windows, vault: `%USERPROFILE%\iCloudDrive\iCloud~md~obsidian\Documents\Brain` (또는 사용자 셋업 위치)
   - 양쪽 vault 경로는 `~/.claude/commands/ingest_config.json`에서 override 가능 (없으면 기본값).

2. **사용자 입력 ($ARGUMENTS) 형태 판별**:
   - `.pdf` 확장자 → PDF ingest 분기 (텍스트 레이어 확인 후 필요시 OCR)
   - `.md` / `.txt` / `.docx` → 텍스트 ingest 분기
   - `http://` 또는 `https://` 시작 → URL 분기 (notion.so → Notion MCP, 일반 → WebFetch)
   - 따옴표·여러 줄 → 직접 paste 분기
   - 폴더 경로 (디렉토리) → 일괄 ingest 분기 (확인 1회)
   - 입력 없음 → 사용자에게 어떤 자료인지 묻기 (1회)

3. **vault CLAUDE.md 핵심 원칙 환기** (필요시만 사용자에게 명시):
   - raw immutable, summary는 같은 폴더에 별도 파일
   - `02-Wiki/` 신규 서브폴더 신설 금지 (concepts/people/frameworks/deals/companies/queries/archive 7개 고정)
   - 한국어 음슴체
   - 실명 회사 + 재무 수치 조합 회피 (단 본인 회사 자체 트랙레코드는 예외)

## STEP 1 — Source 본문 확보

| 형태 | 처리 |
|---|---|
| **텍스트 레이어 있는 PDF** | `python3 -c "import fitz; d=fitz.open('<path>'); print('\\n'.join(d[i].get_text() for i in range(d.page_count)))"` |
| **스캔 PDF (이미지)** | pymupdf로 3-4페이지×N청크 분할(120 DPI JPEG 압축, ~400KB/청크) → Read 도구 multimodal OCR 순차 호출. 20페이지 넘으면 서브에이전트(general-purpose) 위임. (Genesis 피치북·다올 Computex 패턴) |
| **Notion URL** | `mcp__claude_ai_Notion__notion-fetch` (UUID 8-4-4-4-12 형태로 변환 필수, 단축 ID는 400 오류) |
| **일반 웹 URL** | WebFetch |
| **마크다운/텍스트 파일** | 직접 Read |
| **paste 텍스트** | 그대로 사용 |

### 📚 BOOK MODE — 읽고 나서 ingest할 때 추가 입력 수집 (source_type=books)

사용자가 **이미 읽은 책**을 ingest할 때는 본문 확보 직후 다음을 확인:

1. **독서 메모·밑줄 있는가?** — 있으면 그대로 붙여달라고 요청 (1회). 없으면 건너뜀.
2. **챕터 목차** — PDF에서 자동 추출, 실패 시 사용자에게 복사 붙여넣기 요청.
3. **읽은 시점 기록** — `read_date: YYYY-MM` frontmatter에 추가.

독서 메모가 있으면 이후 모든 단계에서 **본문 원문과 메모를 병렬로 참조**해 사용자 관점이 summary에 스며들게 할 것.

## STEP 2 — source_type 자동 추정 (CLAUDE.md §2.1.1)

- URL 포함 → `articles`
- "책", "챕터", "p.XX" 언급 → `books`
- "영상", "분", "유튜브" → `videos`
- "팟캐스트", "EP." → `podcasts`
- "딜", "IM", "티저", 코드네임, 펀드 피치북, 트랙레코드 → `deals`
- "미팅", 참석자 언급 → `meetings`
- PE Research Agent Daily Research → `articles` (vault 관례)
- 애매 → `articles`

## STEP 3 — 파일 생성 (raw + summary)

**파일명 규칙** (CLAUDE.md §4):
- `01-Sources/<type>/YYYY-MM-DD-원제목.md` (raw)
- `01-Sources/<type>/YYYY-MM-DD-원제목 - 요약.md` (summary)
- 한국어 제목 우선, 공백 그대로, 특수문자만 `-` 치환
- **`02-Wiki/`에 source summary 폴더 신설 금지** ([[feedback-brain-vault-source-location]])

**raw 파일 frontmatter** (immutable이지만 frontmatter는 추가 OK):
```yaml
---
type: source
source_type: article | deal | book | ...
created: YYYY-MM-DD
updated: YYYY-MM-DD
read_date: YYYY-MM          # books: 읽은 연월
tags: [도메인/투자/PE, 타입/아티클, 상태/active]
title: "원제목"
author: "작성자"
원본_경로: "<absolute path or URL>"
status: active
related: "[[YYYY-MM-DD-... - 요약]]"
---
```

**summary 파일 frontmatter + 본문** (§6.1 뼈대):
- frontmatter: `raw_file`, `related`, 우선순위/높음 등 적정 태그
- 본문: 핵심 요약 → 주요 Takeaways → 언급된 개념/인물 → 본인 의견·반대 의견 → 연결·추가 탐색

**큰 raw (PDF 50MB+)**: vault 복사 vs 외부 참조 결정. 단일 자산이고 가치 높으면 vault 내장 OK (Genesis 피치북 108MB 선례), 23GB급은 외부 참조만 (SV 트랙레코드 선례).

---

### 📚 BOOK 전용 summary 템플릿 (source_type=books)

일반 summary보다 훨씬 풍부하게 작성. 아래 섹션을 모두 포함:

```markdown
## 한 줄 핵심
<저자가 책 전체로 말하려는 단 하나의 주장>

## 저자 프레임워크 / 멘탈모델
<이 책이 제시하는 핵심 사고 틀, 개념 구조, 원칙 체계>
- 모델명: 설명
- 모델명: 설명
(재사용 가능한 추상 개념 위주. wiki concepts/ 승급 후보)

## 챕터별 핵심
| 챕터 | 핵심 주장 | 기억할 것 |
|---|---|---|
| 1. 제목 | | |
| 2. 제목 | | |
...

## 주요 인용구
> "직접 인용 1" — p.XX
> "직접 인용 2" — p.XX
(3~10개. 다른 문서에서 재인용할 만한 것 위주)

## 사용자 독서 메모 통합
<사용자가 붙여준 밑줄·메모를 원문 맥락에 연결해 서술>

## 주요 Takeaways
1. ...
2. ...

## 언급된 개념·인물·회사
[[개념1]], [[개념2]], [[인물1]], [[회사1]] ...

## 본인 의견 / 비판적 시각
<동의하는 점, 의문점, 반대 의견>

## SVI 시사점
<PE 운용 실무 관점에서의 적용 포인트>

## 연결·추가 탐색
- 연결 wiki 페이지: [[...]]
- 다음에 읽을 책: ...
- 검증 필요 주장: ...
```

## STEP 4 — 기존 wiki 페이지 append (가장 중요)

본문에 언급된 개념·인물·회사를 점검해서 기존 페이지에 wikilink·내용 append. **링크 밀도가 vault 가치의 핵심**.

**점검 절차**:
1. 본문에서 후보 추출 (PE 운용사·회사·concept·person 명사)
2. vault에 동일 페이지 존재 여부:
   ```bash
   grep -rl "^# 회사명$" "$VAULT/02-Wiki/" 2>/dev/null
   ```
3. 존재 → 해당 페이지 frontmatter `sources`에 본 source 추가 + 본문에 새 섹션·표 행 append
4. 부재 → wikilink만 박고 (`[[회사명]]`) lint 큐에 자동 노출

**watchlist 6사 ([[리벨리온]]·[[삼성전자]]·[[NVIDIA]]·[[Tesla]]·[[Palantir]]·[[엘앤씨바이오]])는 「누적 관찰」 표에 한 행 append**.

**📚 BOOK 추가 규칙**:
- 챕터별 핵심 테이블에서 추출한 개념 후보 **모두** 점검 (articles보다 범위 넓게)
- 인용구 섹션의 인물명 → `02-Wiki/people/`에 wikilink 삽입
- 저자 프레임워크의 모델명 → `02-Wiki/frameworks/` 또는 `concepts/`에 연결
- 기존 wiki 페이지에 **"이 책에서 언급됨"** 행 추가 시 인용구 1개도 함께 붙임

**검증분/정정분 흡수 패턴** (v1→v2→v3→v4 같은 경우, [[feedback-brain-vault-ingest-workflow]] 참조):
- 기존 v1 페이지 frontmatter에 `vN_verification`/`vN_primary_source`/`vN_pe_agent` 키 추가
- 상단에 ⚠️ 정정/업데이트 배너
- 영향받는 섹션에 🆕 인라인 정정 표기
- 기존 태그(`상태/확인필요`) 제거 X — `상태/검증완료-부분`/`상태/검증완료` 병기

## STEP 5 — 신규 wiki page 생성 여부 판단 (§6.7 승급 규율)

- **company**: daily briefing 등에서 3회+ 누적 등장한 watchlist 종목만 승급. 1-2회는 stub link만 (dead-link로 lint 큐에 표시)
- **concept**: 재사용 가능한 추상 아이디어, 다른 source에서 재인용 가능성 — 1회 사례면 보류, 2-3건 누적 시 승급
- **deal**: 회수 완료된 케이스 스터디 또는 진행 중 활성 딜

**page 양산 자제 원칙**: 1건 source ingest 시 신규 wiki page는 0~2개 권장. 251사 SV 트랙레코드처럼 큰 dataset도 per-company 페이지 X.

**📚 BOOK 완화 규칙** (books는 더 관대하게 승급):
- **frameworks/**: 저자 고유 프레임워크·멘탈모델 → **1회 등장해도 즉시 승급** (책이 primary source이므로)
- **concepts/**: 책에서 상세히 정의된 개념 → 1회도 OK, 단 정의·설명·출처 명시 필수
- **people/**: 책에서 핵심 인물로 다뤄지는 사람 → 승급 (단순 언급은 wikilink만)
- **1권 ingest 시 신규 wiki page 0~4개** (일반 source 0~2개보다 완화)
- 승급 기준: "다른 source를 읽다가 이 페이지를 다시 참조할 것 같은가?"

## STEP 6 — `00-Meta/index.md` 갱신

1. 상단 "마지막 업데이트" 한 줄에 본 ingest 요약 추가, 직전 항목은 "이전:" 아래로 밀어내기
2. 통계 카운트 업데이트 (Total Sources, Wiki Pages, etc.)
3. 해당 카테고리 (Articles/Books/Deals/Meetings/Videos) 섹션에 신규 항목 추가 (한 줄 요약 + 핵심 facts)

## STEP 7 — `00-Meta/log.md` append

새 `## [YYYY-MM-DD] ingest | 제목` 헤더 + 본문:
- 원본 경로·source_type·우선순위
- ingest 방식 (OCR·서브에이전트 사용 시 명시)
- 생성 파일 목록
- 기존 페이지 append 목록
- 핵심 발견 (3-7개 bullet)
- 판단 메모 (양산·승급 결정 근거)
- SVI 시사점
- Index 갱신 카운트 변화
- 다음 액션 후보 (3-5개)
- 끝에 `---` 구분선

## STEP 8 — 대시보드 재빌드 (macOS만)

```bash
[ -f ~/brain-dashboard/build.py ] && python3 ~/brain-dashboard/build.py 2>&1 | tail -6
```
Windows는 launchd 미설치라 수동 — 사용자에게 안내만.

## STEP 9 — 결과 보고

사용자에게 보고 형식 (한국어 음슴체):
- ✅ 생성 파일 (raw + summary 파일명)
- 🔗 기존 페이지 append (페이지명 + 핵심 변경점)
- 🆕 신규 wiki page (있으면)
- 핵심 발견 3-5개
- ⚠️ 검증 필요 사항
- 다음 액션 후보 (사용자 결정 필요한 것만)

## 누락 금지 체크

- [ ] raw `01-Sources/<type>/`에만 있는가
- [ ] summary frontmatter `type: source` + `raw_file` 키 있는가
- [ ] wikilink `[[...]]` 10개+ 본문에 박혔는가
- [ ] index.md 통계·해당 섹션 둘 다 갱신했는가
- [ ] log.md 새 entry 끝에 `---` 구분선 있는가
- [ ] 신규 폴더 만들지 않았는가 (7개 고정)
- [ ] 응답은 한국어 음슴체인가
- [ ] (macOS) 대시보드 재빌드 했는가

**📚 BOOK 추가 체크**:
- [ ] 챕터별 핵심 테이블 작성했는가
- [ ] 저자 프레임워크/멘탈모델 섹션 있는가
- [ ] 인용구 3개+ 추출했는가 (p.XX 포함)
- [ ] 사용자 독서 메모가 있었다면 통합했는가
- [ ] `read_date: YYYY-MM` frontmatter 있는가
- [ ] 프레임워크·핵심 개념 wiki 승급 검토했는가

---

## Karpathy LLM Wiki 패턴 (운영 참고)

Karpathy(2026.04)가 정의한 3-오퍼레이션:
- **Ingest** (본 skill): 소스 → raw 보존 → wiki 업데이트 (현재 구현됨)
- **Query**: wiki 기반 질문 → 가치 있는 답변은 새 wiki 페이지로 저장 ("탐색이 지식이 된다")
- **Lint**: 주기적 헬스체크 — 고아 페이지, 모순, 오래된 정보 탐지 (`/lint-wiki` 커맨드 예정)

**index.md는 에이전트의 나침반**: Query 시 항상 index.md부터 읽어야 관련 파일을 빠르게 탐색 가능.

## 사용 예시

```
/ingest C:\Users\aerok86\Downloads\report.pdf
/ingest https://www.notion.so/690d6d6446dc4f4fa9700543377139c2
/ingest 다음 텍스트 ingest 해줘: <paste>
/ingest C:\Users\aerok86\Downloads\reports_folder\
```

## 입력 인자

$ARGUMENTS

위 입력을 STEP 0부터 순서대로 진행하라. 환경(macOS·Windows)을 먼저 확인하고, vault 경로 차이만 변수로 처리. 나머지 워크플로우는 OS 무관.
