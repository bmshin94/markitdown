# MarkItDown 전수조사 & 수익화 Q&A 정리 💖

> 이 문서는 MarkItDown 저장소를 **전수조사(소스코드 직접 분석)** 한 뒤,
> 설치·사용법 / 정체성(플러그인·스킬·MCP) / API 토큰 / AI 에이전트 활용 /
> 수익화 / React·PHP 연동 / 유튜브 강의 제작 가능성까지 정리한 대화 요약본입니다.
>
> 작성: 카리나 (Claude Code) · 날짜: 2026-10-06

---

## 📎 관련 주소 모음

| 구분 | 주소 |
|---|---|
| **이 저장소 (fork)** | https://github.com/bmshin94/markitdown |
| 작업 브랜치 | https://github.com/bmshin94/markitdown/tree/claude/epic-albattani-h8gqkx |
| 원본 (Microsoft) | https://github.com/microsoft/markitdown |
| PyPI — 본체 | https://pypi.org/project/markitdown/ |
| PyPI — MCP 서버 | https://pypi.org/project/markitdown-mcp/ |
| PyPI — OCR 플러그인 | https://pypi.org/project/markitdown-ocr/ |
| 이슈 트래커 | https://github.com/microsoft/markitdown/issues |
| 기여 가능 이슈 | https://github.com/microsoft/markitdown/issues?q=is%3Aissue+is%3Aopen+label%3A%22open+for+contribution%22 |
| MCP 공식 문서 | https://modelcontextprotocol.io/ |
| Azure Document Intelligence | https://learn.microsoft.com/azure/ai-services/document-intelligence/ |
| Azure Content Understanding | https://learn.microsoft.com/azure/ai-services/content-understanding/ |
| 함께 보기 (상세 스터디 노트) | [`MARKITDOWN_STUDY_KR.md`](./MARKITDOWN_STUDY_KR.md) |

---

## 목차

1. [프로젝트 정체 — 전수조사 결과](#1-프로젝트-정체--전수조사-결과)
2. [폴더 구조 전수조사](#2-폴더-구조-전수조사)
3. [동작 원리](#3-동작-원리)
4. [쉬운 비유로 이해하기](#4-쉬운-비유로-이해하기)
5. [설치 및 사용법](#5-설치-및-사용법)
6. [플러그인? 스킬? MCP?](#6-플러그인-스킬-mcp)
7. [API 토큰이 필요한가](#7-api-토큰이-필요한가)
8. [AI 에이전트 구축에 도움이 되는가](#8-ai-에이전트-구축에-도움이-되는가)
9. [React / PHP 연동 방법](#9-react--php-연동-방법)
10. [유튜브 강의 제작 가능성](#10-유튜브-강의-제작-가능성)
11. [수익화 아이디어 10선](#11-수익화-아이디어-10선)
12. [추천 로드맵](#12-추천-로드맵)
13. [사업 시작 전 체크리스트](#13-사업-시작-전-체크리스트)

---

## 1. 프로젝트 정체 — 전수조사 결과

**한 줄 요약: 어떤 파일이든(PDF·워드·엑셀·PPT·이미지·음성·유튜브) LLM이 읽기 좋은 마크다운으로 바꿔주는 변환기.**

| 항목 | 내용 |
|---|---|
| 프로젝트명 | MarkItDown |
| 만든 곳 | Microsoft (AutoGen 팀) |
| 라이선스 | **MIT** (상업적 이용·수정·재배포·판매 허용) |
| 언어 / 요구 버전 | Python **3.10 이상** |
| 버전 | `0.1.8b1` (베타, `packages/markitdown/src/markitdown/__about__.py`) |
| 코드 규모 | `.py` 합계 약 **15,850줄** |
| 구조 | 모노레포 — `packages/` 아래 **4개 패키지** |
| 변환기 수 | **22개** (`converters/__init__.py`) |
| CI | GitHub Actions 2개 (`tests.yml`, `pre-commit.yml`) |

### 지원 포맷 (변환기 22개 전체)

```
문서     : PDF, DOCX, PPTX, XLSX, XLS, EPUB, Outlook .msg, Jupyter .ipynb
텍스트   : PlainText, CSV, HTML
웹 특화  : Wikipedia, YouTube(자막), Bing SERP, RSS/Atom
미디어   : 이미지(EXIF + LLM 설명), 오디오(EXIF + 음성인식)
컨테이너 : ZIP (내부를 재귀적으로 전부 변환)
클라우드 : Azure Document Intelligence, Azure Content Understanding
```

### 왜 마크다운인가

- 제목·표·리스트 등 **문서 구조 보존**
- HTML 대비 **토큰 효율이 좋음** → LLM 비용 절감
- 주요 LLM이 마크다운으로 대량 학습되어 **자연스럽게 이해**

### 어떨 때 쓰는가

1. **RAG 챗봇 전처리** (가장 흔한 용도)
2. **AI 에이전트의 "눈"** — MCP로 연결
3. 사내 문서 **일괄 마이그레이션** (워드 → 마크다운 위키)
4. 데이터 수집 파이프라인 (유튜브 자막·RSS·위키)
5. **토큰 비용 절감**

---

## 2. 폴더 구조 전수조사

```
markitdown/
├── CLAUDE.md                     # 페르소나 설정
├── MARKITDOWN_STUDY_KR.md        # 한국어 상세 스터디 노트 (38KB)
├── MARKITDOWN_QA_SUMMARY_KR.md   # ← 이 문서
├── README.md                     # 공식 문서 (17KB)
├── LICENSE                       # MIT
├── SECURITY.md / SUPPORT.md / CODE_OF_CONDUCT.md
├── Dockerfile / .devcontainer/
├── .github/workflows/            # tests.yml, pre-commit.yml
└── packages/
    ├── markitdown/               # ⭐ 본체 (라이브러리 + CLI)
    ├── markitdown-mcp/           # MCP 서버
    ├── markitdown-ocr/           # OCR 플러그인 (LLM Vision)
    └── markitdown-sample-plugin/ # 플러그인 작성 예제 (RTF)
```

### 2-1. `packages/markitdown` — 본체

| 파일 | 역할 |
|---|---|
| `_markitdown.py` (32KB) | 심장부. `MarkItDown` 클래스 — 변환기 등록·타입 추론·분기 |
| `_base_converter.py` | 모든 변환기의 부모. `accepts()` + `convert()` 2개 메서드만 |
| `_stream_info.py` | 파일 정보 꾸러미 (mimetype / extension / charset / filename / url / local_path) |
| `__main__.py` (9KB) | CLI 진입점 (`markitdown` 명령) |
| `_exceptions.py` | `MissingDependencyException`, `FileConversionException` 등 |
| `converters/` | 변환기 22개 |
| `converter_utils/docx/math/` | 워드 수식(OMML) → LaTeX 변환 유틸 |
| `tests/` | 테스트 21개 파일 + 샘플 파일 40여 개 |

### 2-2. `packages/markitdown-mcp` — MCP 서버

- `FastMCP` 기반, 코드 **140줄**로 매우 가벼움
- 노출 도구는 **1개**: `convert_to_markdown(uri)` — `http:`, `https:`, `file:`, `data:` 지원
- 전송 방식 3가지: **STDIO**(기본) / **Streamable HTTP** / **SSE**
- 기본 바인딩 `127.0.0.1` + **인증 없음** → 외부 인터페이스 바인딩 금지 ⚠️
- `MARKITDOWN_ENABLE_PLUGINS=true` 환경변수로 플러그인 활성화

### 2-3. `packages/markitdown-ocr` — OCR 플러그인

- PDF / DOCX / PPTX / XLSX **내부 이미지의 글자**를 추출
- Tesseract 등 바이너리 **불필요** — `llm_client`(OpenAI 호환) + Vision 모델이 OCR 담당
- 스캔 PDF는 페이지 전체를 이미지로 떠서 OCR 하는 **full-page fallback** 보유
- `llm_client` 미제공 시 조용히 건너뛰고 기본 변환기로 폴백 (안전 설계)

### 2-4. `packages/markitdown-sample-plugin` — 플러그인 교본

RTF 변환기 하나로 플러그인 작성법을 보여주는 샘플. 규칙 3개:

```python
__plugin_interface_version__ = 1              # ① 인터페이스 버전
def register_converters(markitdown, **kwargs): # ② 등록 함수
    markitdown.register_converter(RtfConverter())
class RtfConverter(DocumentConverter):         # ③ accepts() + convert()
    ...
```

`pyproject.toml`:
```toml
[project.entry-points."markitdown.plugin"]
my_plugin = "my_package._plugin"
```

---

## 3. 동작 원리

```
파일 / URL 입력
   ↓
① magika(구글 AI 파일탐지기) + 확장자 + MIME + content-disposition
   → "StreamInfo 추측 리스트" 생성
   ↓
② 등록된 변환기에 순서대로 accepts() 질의
   - PRIORITY_SPECIFIC_FILE_FORMAT (0)  = 특정 포맷 전용(PDF, DOCX…) → 먼저
   - PRIORITY_GENERIC_FILE_FORMAT (10)  = 범용(PlainText, HTML, Zip) → 나중
   ↓
③ True를 반환한 변환기의 convert() 실행
   ↓
④ DocumentConverterResult(markdown=..., title=...) 반환
   ↓
   실패 시 다음 변환기로, 전부 실패하면 FailedConversionAttempt 목록과 함께 에러
```

### 인상적인 설계 디테일

- `DocxConverter`는 `HtmlConverter`를 **상속** → 워드 → (mammoth) HTML → 마크다운 2단 변환
- HTTP 요청 시 `Accept: text/markdown, text/html;q=0.9, …` 헤더를 먼저 보냄
  (Cloudflare의 "markdown for agents" 대응)
- 확장자를 속여도 `magika`가 내용으로 실제 타입을 판별
- 변환기 추가가 매우 쉬움 — 클래스 하나 + 메서드 2개

---

## 4. 쉬운 비유로 이해하기

### 비유 1 — 만능 통역사

```
📄 PDF  ─┐
📊 엑셀 ─┤
📝 워드 ─┼→ 🎤 MarkItDown ─→ 마크다운 텍스트 ─→ 🤖 LLM
🎧 녹음 ─┤      (통역사)
📺 유튜브─┘
```

LLM은 "텍스트"만 알아듣는다. PDF는 바이너리 덩어리라서 그대로는 못 읽는다.

### 비유 2 — 왜 마크다운인가

| 번역 언어 | 장점 | 단점 |
|---|---|---|
| 그냥 텍스트 | 가볍다 | 제목/표 구분이 사라진다 |
| HTML | 구조 보존 | 태그가 토큰을 많이 먹는다 (비용 ↑) |
| **마크다운** | 구조 보존 + 가볍다 | — |

### 비유 3 — 내부 구조는 "병원 접수대"

```
① 접수   : magika가 "이 환자(파일)의 실제 정체"를 판별
② 배정   : 22명의 전문의에게 "볼 수 있나요?" 질의 (accepts)
           전문의(PDF·DOCX) 먼저 → 못 보면 일반의(PlainText)
③ 진료   : 담당 변환기가 convert() 수행
④ 실패   : 다음 전문의 → 전부 실패하면 에러로 보고
```

### 비유 4 — 4개 패키지는 "본체 + 액세서리"

```
📦 markitdown            본체
🔌 markitdown-mcp        AI와 연결하는 케이블
🔍 markitdown-ocr        문서 속 이미지를 읽는 돋보기
📖 sample-plugin         액세서리 제작 설명서
```

### 비유 5 — 로컬 vs 클라우드

| 방식 | 비유 | 품질 | 비용 |
|---|---|---|---|
| 기본 변환기 | 집에서 요리 | 보통 | 무료 |
| Azure Doc Intelligence | 배달 | 좋음 (스캔 PDF·복잡한 표) | 호출당 과금 |
| Azure Content Understanding | 미슐랭 | 최고 (**영상·음성** + 필드 추출) | 호출당 과금 |

> 영상 파일은 **Content Understanding이 유일한 선택** — 기본 변환기는 영상 미지원.

---

## 5. 설치 및 사용법

### 5-1. 설치

```bash
# 0) 파이썬 3.10+ 확인
python3 --version

# 1) 가상환경 (권장)
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 2-A) 일반 설치 (전체 기능)
pip install 'markitdown[all]'

# 2-B) 필요한 것만
pip install 'markitdown[pdf,docx,xlsx,pptx]'

# 2-C) 이 저장소 소스로 (편집 모드)
pip install -e 'packages/markitdown[all]'

# 3) 확인
markitdown --version
markitdown --list-plugins
```

### 5-2. 선택 의존성 목록 (`pyproject.toml` 기준)

| 옵션 | 설치 패키지 | 용도 |
|---|---|---|
| `[all]` | 전부 | 전체 기능 |
| `[pdf]` | pdfminer.six, pdfplumber | PDF |
| `[docx]` | mammoth, lxml | 워드 |
| `[xlsx]` | pandas, openpyxl | 엑셀 |
| `[xls]` | pandas, xlrd | 구버전 엑셀 |
| `[pptx]` | python-pptx | 파워포인트 |
| `[outlook]` | olefile | Outlook .msg |
| `[audio-transcription]` | pydub, SpeechRecognition | 음성 → 텍스트 |
| `[youtube-transcription]` | youtube-transcript-api | 유튜브 자막 |
| `[az-doc-intel]` | azure-ai-documentintelligence, azure-identity | Azure 문서 AI |
| `[az-content-understanding]` | azure-ai-contentunderstanding, azure-identity | Azure 멀티모달 |

### 5-3. CLI

```bash
markitdown 문서.pdf                 # 화면 출력
markitdown 문서.pdf -o 결과.md       # 파일 저장
markitdown 문서.pdf > 결과.md        # 리다이렉트
cat 문서.pdf | markitdown           # 파이프 (stdin)

# 폴더 일괄 변환
for f in *.pdf; do markitdown "$f" -o "${f%.pdf}.md"; done
```

**전체 옵션** (`__main__.py` 기준)

| 옵션 | 설명 |
|---|---|
| `-o, --output` | 출력 파일 |
| `-x, --extension` | 확장자 힌트 (stdin 사용 시 중요) |
| `-m, --mime-type` | MIME 힌트 |
| `-c, --charset` | 인코딩 힌트 (예: `cp949`) |
| `-p, --use-plugins` | 플러그인 활성화 |
| `--list-plugins` | 설치된 플러그인 목록 |
| `--keep-data-uris` | base64 이미지 유지 (기본은 절삭) |
| `-d, --use-docintel` | Azure Document Intelligence 사용 |
| `-e, --endpoint` | Azure DI 엔드포인트 (`MARKITDOWN_DOCINTEL_ENDPOINT`) |
| `--use-cu` | Azure Content Understanding 사용 |
| `--cu-endpoint` | CU 엔드포인트 (`MARKITDOWN_CU_ENDPOINT`) |
| `--cu-analyzer` | CU 분석기 ID (커스텀 필드 추출) |
| `--cu-file-types` | CU로 보낼 파일 타입 제한 (비용 통제) |
| `-v, --version` | 버전 |

**힌트가 필요한 순간**
```bash
cat 파일 | markitdown -x .pdf     # stdin은 파일명이 없음
markitdown data.csv -c cp949      # 엑셀에서 저장한 한글 CSV
```

### 5-4. Python API

```python
from markitdown import MarkItDown

md = MarkItDown()                 # 플러그인 비활성(기본)
result = md.convert("파일.pdf")
print(result.markdown)            # 변환 결과
print(result.title)               # 제목(있으면)
```

**convert 계열 함수 — 보안상 반드시 구분해서 사용**

| 함수 | 입력 | 보안 |
|---|---|---|
| `convert()` | 경로 / URL / 스트림 전부 | 🔴 가장 허용적 (위험) |
| `convert_local(path)` | 로컬 파일만 | 🟡 |
| `convert_uri(uri)` | URI만 | 🟡 |
| `convert_stream(stream)` | 바이트 스트림만 | 🟢 가장 안전 |
| `convert_response(resp)` | requests 응답 | 🟢 |

```python
# ❌ 위험: 사용자 입력을 그대로 — file:///etc/passwd 등으로 서버 파일 유출 가능
md.convert(user_input)

# ✅ 안전
with open(safe_path, "rb") as f:
    result = md.convert_stream(f)
```

**LLM 이미지 설명**
```python
from openai import OpenAI
md = MarkItDown(llm_client=OpenAI(), llm_model="gpt-4o")
print(md.convert("사진.jpg").markdown)
```

**Docker**
```bash
docker build -t markitdown:latest .
docker run --rm -i markitdown:latest < 파일.pdf > 결과.md
```

---

## 6. 플러그인? 스킬? MCP?

**결론: 본질은 "파이썬 라이브러리", 그 위에 CLI·MCP·플러그인 호스트가 함께 제공된다.**

| 정체성 | 해당 | 설명 |
|---|---|---|
| ① 파이썬 라이브러리 | ⭐ **본질** | `pip install markitdown` → `import markitdown` |
| ② CLI 도구 | ✅ | `markitdown file.pdf` (`[project.scripts]` 등록) |
| ③ MCP 서버 | ✅ | `packages/markitdown-mcp` |
| ④ 플러그인 **호스트** | ✅ | `entry_points`(group=`markitdown.plugin`)로 3rd-party 수용 |
| ⑤ Claude Skill | ❌ | 기본 제공 아님 (직접 Skill로 감싸는 건 가능) |

```
MarkItDown 자체   = 라이브러리
markitdown-mcp    = AI에게 읽어주는 사서 (MCP 서버)
markitdown-ocr    = 끼워 넣는 돋보기 (플러그인)
Claude Skill      = 직접 만들면 가능 (현재 없음)
```

### MCP 연결 — Claude Desktop

`claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "markitdown": {
      "command": "docker",
      "args": ["run", "--rm", "-i",
               "-v", "/Users/me/docs:/workdir",
               "markitdown-mcp:latest"]
    }
  }
}
```

### MCP 연결 — Claude Code

```bash
claude mcp add markitdown -- markitdown-mcp
```

### 디버깅

```bash
npx @modelcontextprotocol/inspector
# STDIO 선택 → markitdown-mcp → Connect → Tools 탭 → convert_to_markdown
```

> 💡 GitHub에 `#markitdown-plugin` 해시태그를 달면 플러그인이 검색된다.
> **한글(HWP) 변환기 플러그인은 아직 사실상 공백 상태 → 기회 영역.**

---

## 7. API 토큰이 필요한가

**결론: 기본 기능은 전부 무료, 토큰 불필요. 선택 기능만 키가 필요하다.**

| 기능 | 토큰/키 | 비용 |
|---|---|---|
| PDF / 워드 / 엑셀 / PPT → MD | ❌ | 무료 |
| HTML / CSV / JSON / XML | ❌ | 무료 |
| ZIP 통째 변환 | ❌ | 무료 |
| EPUB / .msg / .ipynb | ❌ | 무료 |
| 이미지 EXIF 메타데이터 | ❌ | 무료 |
| 유튜브 자막 / 위키 / RSS | ❌ | 무료 |
| **이미지 AI 설명** | ✅ OpenAI 등 | 호출당 과금 |
| **OCR 플러그인** | ✅ OpenAI 등 | 호출당 과금 |
| 음성 → 텍스트 | △ 기본은 Google 무료 웹 API | 사실상 무료(불안정) |
| **Azure Doc Intelligence** | ✅ Azure | 페이지당 과금 |
| **Azure Content Understanding** | ✅ Azure | 호출당 과금 |

### 비용 통제

README 경고: **CU로 라우팅되는 포맷의 `convert()` 호출 하나하나가 과금 대상.**

```python
from markitdown.converters import ContentUnderstandingFileType
md = MarkItDown(cu_endpoint="...",
                cu_file_types=[ContentUnderstandingFileType.PDF])  # PDF만 CU
```

### 환경변수로 관리 (하드코딩 금지)

```bash
export OPENAI_API_KEY="..."
export MARKITDOWN_DOCINTEL_ENDPOINT="https://..."
export MARKITDOWN_CU_ENDPOINT="https://..."
export MARKITDOWN_ENABLE_PLUGINS="true"
```

---

## 8. AI 에이전트 구축에 도움이 되는가

**결론: 매우 도움이 된다. 사실상 필수 구성요소.**

```
🧠 두뇌 = LLM      👂 귀 = 사용자 입력     ✋ 손 = 도구 호출
👀 눈  = MarkItDown  💾 기억 = 벡터DB
```

### 활용 시나리오

**① RAG 파이프라인 1단계**
```
사내 문서(PDF·엑셀 수백 개) → MarkItDown → 청킹 → 임베딩 → 벡터DB → 검색 → LLM
```
포맷별 파서를 직접 짜지 않아도 22개 포맷을 확보.

**② LangChain / LlamaIndex 연동**
```python
from markitdown import MarkItDown
from langchain.text_splitter import MarkdownTextSplitter

text = MarkItDown().convert("보고서.pdf").markdown
chunks = MarkdownTextSplitter(chunk_size=1000).split_text(text)
```
마크다운이라 `#` 제목 기준으로 **의미 단위를 깨지 않고** 청킹 가능 → RAG 품질 ↑

**③ 에이전트 도구로 등록**
```python
from langchain.tools import tool

@tool
def read_document(path: str) -> str:
    """로컬 문서를 읽어 마크다운으로 반환한다."""
    return MarkItDown().convert_local(path).markdown   # convert_local 사용
```

**④ MCP로 Claude에 직접 연결** — 코드 0줄, 설정만

**⑤ 멀티모달 에이전트** — Content Understanding으로 영상·음성까지

### 한계와 대응

| 한계 | 대응 |
|---|---|
| 복잡한 표·다단 레이아웃 PDF 깨짐 | Azure Doc Intel / CU 병행 |
| 스캔 PDF는 기본 변환기로 불가 | `markitdown-ocr` 플러그인 |
| 이미지 많은 문서 → LLM 비용 | 필요한 파일만 OCR |
| 비동기(async) API 없음 | ThreadPool / 워커로 감싸기 |
| `convert()`가 너무 허용적 | `convert_stream()` / `convert_local()` 사용 |
| 버전 `0.1.8b1` = 베타 | 프로덕션은 버전 고정(pin) |

---

## 9. React / PHP 연동 방법

**결론: 재구현은 비효율. "연동"이 정답.**

### 9-1. 재구현이 비효율인 이유

| 기능 | 파이썬 | JS / PHP 대안 |
|---|---|---|
| 파일타입 탐지 | **magika** (구글 AI 모델) | 동급 없음 |
| PDF | pdfminer.six + pdfplumber | pdf-parse (표 추출 약함) |
| 워드 | **mammoth** (스타일맵 포함) | mammoth.js (기능 축소) |
| 엑셀 | pandas + openpyxl | SheetJS / PhpSpreadsheet |
| 워드 수식 OMML→LaTeX | 전용 유틸 | 거의 없음 |
| 음성 인식 | SpeechRecognition | 없음 |

### 9-2. 권장 구조

```
[React 프론트]           UI·업로드·미리보기
      ↕ REST
[PHP / Node 백엔드]      인증·과금·DB·파일검증
      ↕ HTTP (내부망)
[Python 변환 서비스]     ★MarkItDown★ (FastAPI)
```

### 9-3. Python 변환 서비스 (FastAPI)

```python
# converter_api.py
from fastapi import FastAPI, UploadFile, HTTPException
from markitdown import MarkItDown, StreamInfo
import io

app = FastAPI()
md = MarkItDown()

MAX_SIZE = 20 * 1024 * 1024
ALLOWED = {".pdf", ".docx", ".xlsx", ".pptx", ".csv", ".html", ".epub"}

@app.post("/convert")
async def convert(file: UploadFile):
    ext = "." + file.filename.rsplit(".", 1)[-1].lower()
    if ext not in ALLOWED:
        raise HTTPException(400, "허용되지 않은 형식")
    data = await file.read()
    if len(data) > MAX_SIZE:
        raise HTTPException(413, "파일이 너무 큽니다")
    result = md.convert_stream(io.BytesIO(data),
                               stream_info=StreamInfo(extension=ext))
    return {"markdown": result.markdown, "title": result.title}
```

```bash
uvicorn converter_api:app --host 127.0.0.1 --port 8001   # 내부망만 노출
```

### 9-4. React

```jsx
function Converter() {
  const [md, setMd] = useState("");
  const [loading, setLoading] = useState(false);

  const upload = async (file) => {
    setLoading(true);
    const fd = new FormData();
    fd.append("file", file);
    const res = await fetch("/api/convert", { method: "POST", body: fd });
    setMd((await res.json()).markdown);
    setLoading(false);
  };

  return (
    <div>
      <input type="file" onChange={(e) => upload(e.target.files[0])} />
      {loading ? <p>변환 중…</p> : <ReactMarkdown>{md}</ReactMarkdown>}
    </div>
  );
}
```

미리보기: `react-markdown` + `remark-gfm`(표) / 편집: `@uiw/react-md-editor`

### 9-5. PHP — 방법 A: CLI 직접 호출 (소규모)

```php
<?php
$ext = strtolower(pathinfo($_FILES['doc']['name'], PATHINFO_EXTENSION));
if (!in_array($ext, ['pdf','docx','xlsx','pptx','csv'])) {
    http_response_code(400); exit('허용되지 않은 형식');
}
$safe = sys_get_temp_dir().'/'.bin2hex(random_bytes(16)).'.'.$ext;
move_uploaded_file($_FILES['doc']['tmp_name'], $safe);

// escapeshellarg 필수 (명령어 삽입 방지)
$markdown = shell_exec('/path/.venv/bin/markitdown '.escapeshellarg($safe).' 2>&1');
unlink($safe);
echo $markdown;
```

> ⚠️ `escapeshellarg()`를 빼면 **원격 명령 실행(RCE) 취약점**이 된다.

### 9-6. PHP — 방법 B: Python API 호출 (권장)

```php
<?php
$ch = curl_init('http://127.0.0.1:8001/convert');
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_TIMEOUT => 120,
    CURLOPT_POSTFIELDS => ['file' => new CURLFile($safe, null, basename($safe))],
]);
$res = json_decode(curl_exec($ch), true);
curl_close($ch);
echo $res['markdown'];
```

### 9-7. 브라우저 단독(서버 없이)?

Pyodide(WASM)로 이론상 가능하지만 `magika`·`pandas` 등 네이티브 의존성 때문에 현실적으로 비권장.
순수 프론트만 원하면 `pdf.js` + `mammoth.js` + `SheetJS` 조합이지만 품질 저하를 감수해야 한다.

### 9-8. 중요 — 원본 저장소의 기여 범위

README 명시:
- **Out of scope**: 웹 서버, REST/HTTP API, 호스팅 변환 서비스, 웹 프론트엔드, 데스크톱/모바일 앱
- 다만 **"별도 패키지·프로젝트로 유지하는 것은 환영"**

→ 웹 서비스를 만드는 것은 환영받는 방향. 단, 원본 저장소 PR로 보내지 말고 **독립 프로젝트**로 운영.

---

## 10. 유튜브 강의 제작 가능성

**결론: 소재로 매우 좋다.**

| 이유 | 설명 |
|---|---|
| 트렌드 적중 | AI / RAG / MCP = 최상위 관심 키워드 |
| 진입장벽 낮음 | 설치 1줄 + 코드 3줄로 결과 확인 → 초반 이탈 적음 |
| 시각적 임팩트 | PDF → 마크다운 변환 장면이 썸네일감 |
| 브랜드 신뢰 | "Microsoft가 공개한" (상표 사용 주의) |
| 시리즈 확장성 | 설치 → 실전 → MCP → 플러그인 → RAG → 배포 |
| 한국어 콘텐츠 희소 | 심화 한국어 강의 거의 없음 |

### 커리큘럼 (8편)

| 편 | 제목(안) | 길이 | 난이도 |
|---|---|---|---|
| 1 | PDF를 AI가 읽게 만드는 3줄 코드 | 8분 | ⭐ |
| 2 | 22개 포맷 전부 변환해보기 (엑셀·PPT·유튜브) | 12분 | ⭐ |
| 3 | Claude에 MCP 연결하기 — AI에게 눈을 달아주다 | 15분 | ⭐⭐ |
| 4 | 스캔 PDF도 읽는다: OCR 플러그인 + GPT-4o | 15분 | ⭐⭐ |
| 5 | 내 전용 플러그인 만들기 (HWP 변환기 도전) | 20분 | ⭐⭐⭐ |
| 6 | RAG 챗봇 만들기 (MarkItDown + LangChain + Chroma) | 25분 | ⭐⭐⭐ |
| 7 | React + FastAPI 문서변환 웹서비스 배포 | 30분 | ⭐⭐⭐ |
| 8 | 소스코드 읽기: 플러그인 아키텍처 분석 | 20분 | ⭐⭐⭐⭐ |

### 제작 팁

1. **Before/After를 10초 안에** — PDF를 LLM에 넣어 실패 → MarkItDown 통과 → 성공
2. **에러 화면도 그대로** — `MissingDependencyException` → `pip install 'markitdown[pdf]'` (신뢰도 ↑)
3. **한국 특화 포인트** — 한글 CSV `-c cp949`, HWP 미지원, 공공기관 PDF (영어권 채널에 없는 차별점)
4. **예제 리포 공개** — 설명란 링크로 구독 전환
5. **상표 주의** — "Microsoft 공식 강의" ❌ / "Microsoft가 공개한 오픈소스 사용법" ✅ / 썸네일에 MS 로고 ❌

### 수익 모델

```
① 애드센스 (느림)
② 유료 강의 연계 (인프런·클래스101)
③ 제휴/스폰서 (벡터DB·클라우드)
④ 자체 SaaS 유입 채널  ← 최고의 조합
```

---

## 11. 수익화 아이디어 10선

### 먼저 — 냉정한 현실

| 항목 | 평가 |
|---|---|
| MarkItDown 자체를 판매 | ❌ 불가능 (무료 오픈소스) |
| 단순 "변환 웹사이트" | ⚠️ 레드오션 (무료 경쟁자 다수) |
| 업종 특화 솔루션 | ✅ 유망 |
| B2B SaaS / API | ✅ 유망 |
| 교육·컨설팅 | ✅ 현금흐름 빠름 |

> **핵심 원칙: 파는 것은 "변환"이 아니라 "변환 이후의 가치".**
> 고객이 원하는 건 마크다운이 아니라 "내 문서로 AI가 답하는 것".

### 라이선스 확인 결과 — 판매 가능 ✅

`LICENSE` 원문:
> "Permission is hereby granted, free of charge … to use, copy, modify, merge, publish, distribute, sublicense, and/or **sell** copies"

**지켜야 할 조건 2개**
1. 저작권 고지 + MIT 라이선스 전문 포함 (사이트 하단/about)
2. **Microsoft 상표 사용 금지** — MS 후원·공식으로 오인될 표현 금지
   (`Powered by MarkItDown (MIT)` 정도는 가능)

추가로 `packages/markitdown/ThirdPartyNotices.md`(12KB)의 하위 의존성 라이선스도 상업 배포 전 확인 필요.

---

### 🥇 1. 업종특화 "문서 → AI 챗봇" SaaS ⭐⭐⭐⭐⭐

```
타깃: 법무법인 / 세무사무소 / 병원 / 학원 / 제조업
문제: 내부 규정·매뉴얼 PDF 수백 개를 직원이 찾아보지 못함
해결: 업로드 → MarkItDown 변환 → 벡터DB → 사내 전용 챗봇
```

| 항목 | 내용 |
|---|---|
| 가격 | 월 29만~99만원 (좌석수 기준) |
| 타깃 | 직원 10~200명 중소기업 |
| 스택 | MarkItDown + FastAPI + Chroma/Pinecone + GPT/Claude + React |
| MVP | 3~4주 |
| 강점 | 업종 특화 프롬프트·용어집이 진짜 moat |
| 리스크 | 보안 요구 높음 → 온프레미스 옵션 필요 |

### 🥈 2. 변환 API (개발자용 B2B) ⭐⭐⭐⭐⭐

| 항목 | 내용 |
|---|---|
| 가격 | Free 100건/월 → $19(5천건) → $99(5만건) → Enterprise |
| 타깃 | AI 스타트업, RAG 개발자, 노코드 유저 |

**반드시 얹어야 할 부가가치**
- 비동기 + 웹훅 (대용량 타임아웃 없음)
- 배치 처리 (ZIP 1,000개 일괄)
- **자동 청킹 + 임베딩까지** (RAG 바로 투입)
- 품질 자동 폴백 (기본 실패 → Azure Doc Intel 전환)
- **한글 HWP/HWPX 지원** ← 국내 최강 무기
- 결과 캐싱

> 🔥 국내 공공기관·학교 문서는 대부분 HWP인데 MarkItDown은 미지원.
> → **"국내 유일 HWP → AI 변환 API"** 포지션 확보 가능.

### 🥉 3. RAG 전처리 파이프라인 패키지 ⭐⭐⭐⭐

```
변환 + 청킹 + 메타데이터 + 중복제거 + 품질점수 + 임베딩 = 올인원
```
오픈코어 모델: 기본 무료 + Pro 월 $49 + 기업 라이선스 500만원
"RAG 품질의 80%는 전처리에서 결정된다"가 세일즈 포인트.

### 4. 한글(HWP) 플러그인 + 유료 지원 ⭐⭐⭐⭐

```
markitdown-hwp 오픈소스 공개 (#markitdown-plugin 으로 명성)
 → 기업 지원·커스터마이징 유료화
```
수익: 기업 기술지원 연 300~1,000만원 / 공공 SI 용역
난이도: HWP 바이너리 파싱이 어려움 → **XML 기반 HWPX부터 시작 권장**

### 5. 지식베이스 마이그레이션 대행 ⭐⭐⭐⭐ (현금흐름 최속)

```
사내 워드/PDF 2,000개 → Notion / Confluence / GitBook 이전
```
가격: 문서당 500~3,000원 또는 프로젝트 500만~3,000만원
장점: **지금 당장 시작 가능**, 선금 구조 / 단점: 노동집약

### 6. 노코드 커넥터 (Zapier / Make / n8n) ⭐⭐⭐

```
Gmail 첨부 PDF 도착 → 자동 변환 → Notion 페이지 생성
```
월 $9~29 저가 다수 모델. 마켓플레이스 노출 = 마케팅 비용 0.

### 7. 버티컬 앱 3종 ⭐⭐⭐

| 앱 | 설명 | 가격 |
|---|---|---|
| 계약서 분석기 | 계약 PDF → 위험조항 자동 탐지 | 건당 1만원 / 월 9.9만원 |
| 영수증·송장 정리기 | 사진·PDF → OCR → 엑셀 회계 데이터 | 월 2.9만원 |
| 논문 독해 도우미 | 논문 PDF → 요약·용어해설·QA | 월 1.9만원 (학생) |

> 참고: 테스트 파일에 `RECEIPT-2024-TXN-98765_retail_purchase.pdf`,
> `MEDRPT-2024-PAT-3847_medical_report_scan.pdf` 등이 이미 존재 → 영수증·의료 문서 대응 실적 있음.

### 8. 교육 콘텐츠 (유튜브 + 유료강의) ⭐⭐⭐

인프런/클래스101 5~15만원, 전자책 2~3만원. SaaS 유입 채널로 최적.

### 9. 기업 온프레미스 구축 + 유지보수 ⭐⭐⭐⭐ (금액 큼)

```
"문서를 외부로 보낼 수 없다" (금융·의료·방산·공공)
 → 사내 서버 설치 + 커스터마이징 + 교육
```
구축 1,000만~5,000만원 + 연 유지보수 20%.
보안 요구가 곧 가격 결정력. 클라우드 SaaS가 진입 못하는 시장.

### 10. 데이터셋 구축 서비스 (B2B) ⭐⭐⭐

"사내 문서로 LLM 파인튜닝" → 변환 + 정제 + QA쌍 생성. 프로젝트당 500만~3,000만원.

---

## 12. 추천 로드맵

```
┌─ 1단계 (0~2개월) 현금 만들기
│   아이디어 5 (마이그레이션 대행) + 8 (유튜브)
│   → 코드 최소로 수익 시작 + 실제 고객 문제 수집
│
├─ 2단계 (2~5개월) 제품화
│   아이디어 2 (변환 API) + HWP/HWPX 플러그인
│   → 1단계에서 자동화한 것을 API로 포장, 국내 독점 포지션 확보
│
└─ 3단계 (5~12개월) 스케일
    아이디어 1 (업종특화 챗봇 SaaS) + 9 (온프레미스)
    → 월 반복수익(MRR) 구조 전환
```

**차별화 지점 3가지**
1. 한글(HWP/HWPX) — 국내 유일 포지션
2. 대용량 비동기 배치 — 개인이 못 하는 영역
3. 업종 특화 도메인 지식 — 기술로 복제 불가한 moat

---

## 13. 사업 시작 전 체크리스트

| 항목 | 내용 |
|---|---|
| ☐ MIT 라이선스 고지 | 사이트 하단 / about 페이지 |
| ☐ Microsoft 상표 미사용 | 로고·"공식" 표현 금지 |
| ☐ `ThirdPartyNotices.md` 확인 | 하위 의존성 라이선스 (12KB) |
| ☐ `convert_stream()` / `convert_local()` 사용 | `convert()` 직접 노출 금지 |
| ☐ 파일 크기·확장자 화이트리스트 | DoS·악성파일 방어 |
| ☐ MCP 서버 외부 노출 금지 | 인증 없음, localhost만 |
| ☐ 버전 고정 | `markitdown==0.1.8b1` (베타, API 변동 가능) |
| ☐ 개인정보 처리방침 | 보관기간·삭제정책 명시 |
| ☐ LLM API 비용 상한 | OCR 남용 방지 |
| ☐ 변환 품질 SLA 고지 | "복잡한 표는 100% 보장 불가" 사전 명시 |

---

## 부록 — 자주 만나는 에러

| 에러 | 원인 | 해결 |
|---|---|---|
| `MissingDependencyException` | 선택 의존성 미설치 | `pip install 'markitdown[pdf]'` 등 |
| `UnsupportedFormatException` | 지원 포맷 아님 | `-x`로 확장자 힌트, 또는 플러그인 작성 |
| 한글 깨짐 (CSV) | cp949 인코딩 | `-c cp949` |
| 스캔 PDF 결과 비어있음 | 이미지라서 텍스트 없음 | `markitdown-ocr` + LLM, 또는 Azure DI |
| stdin 변환 실패 | 파일 타입 추론 실패 | `-x .pdf` 힌트 |
| 유튜브 자막 실패 | 자막 없음 / 비공식 API 변동 | `[youtube-transcription]` 설치 확인 |
| ffmpeg 경고 | pydub가 ffmpeg 못 찾음 | ffmpeg 설치 |

---

## 요약 한 줄

> **MarkItDown은 무료·MIT의 "문서 → 마크다운 통역사"다.
> 이 도구 자체로는 돈이 되지 않지만, 이것을 감싼 "업종 특화 솔루션 + 한글(HWP) 대응 +
> 대용량 자동화"는 충분히 수익화 가능한 영역이다.** 💖

---

### 📎 저장소: https://github.com/bmshin94/markitdown
