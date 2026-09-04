# MarkItDown 스터디 노트 🇰🇷

> 이 저장소를 처음 받아본 사람을 위한 한국어 분석 문서입니다.
> 프로젝트가 무엇인지, 어떻게 쓰는지, 수익화가 가능한지, PHP에서 연동할 수 있는지를 정리했습니다.

## 📎 저장소 주소

| 구분 | 주소 |
|---|---|
| 이 저장소 (fork) | https://github.com/bmshin94/markitdown |
| 원본 (Microsoft) | https://github.com/microsoft/markitdown |
| PyPI - 본체 | https://pypi.org/project/markitdown/ |
| PyPI - MCP 서버 | https://pypi.org/project/markitdown-mcp/ |
| 이슈 트래커 | https://github.com/microsoft/markitdown/issues |

---

## 목차

1. [MarkItDown이란?](#1-markitdown이란)
2. [폴더 구조 분석](#2-폴더-구조-분석)
3. [설치 방법](#3-설치-방법)
4. [사용법 - CLI](#4-사용법---cli)
5. [사용법 - Python](#5-사용법---python)
6. [Claude 연동 (MCP)](#6-claude-연동-mcp)
7. [OCR 플러그인](#7-ocr-플러그인)
8. [자주 만나는 에러](#8-자주-만나는-에러)
9. [수익화 검토](#9-수익화-검토)
10. [PHP 연동 방법](#10-php-연동-방법)
11. [보안 체크리스트](#11-보안-체크리스트)

---

## 1. MarkItDown이란?

**한 줄 요약: 어떤 파일이든 LLM이 읽기 좋은 마크다운으로 바꿔주는 변환 도구.**

Microsoft AutoGen 팀이 만든 파이썬 유틸리티입니다.

### 왜 필요한가

LLM(GPT, Claude 등)은 본질적으로 **텍스트만** 읽을 수 있습니다. PDF·워드·엑셀은 그냥 알 수 없는 바이너리 덩어리로 보입니다.

```
📄 PDF ─┐
📊 엑셀 ─┤
🎤 녹음 ─┼─→ [MarkItDown] ─→ 📝 마크다운 텍스트 ─→ 🤖 LLM
📺 유튜브 ─┤
🖼️ 이미지 ─┘
```

### 왜 하필 마크다운인가

README에 나온 이유:

- 제목/표/리스트 등 **문서 구조가 보존**된다
- HTML처럼 태그가 덕지덕지 붙지 않아 **토큰 효율이 좋다** (= 비용 절감)
- 주요 LLM들이 마크다운으로 학습되어 있어 **자연스럽게 이해**한다

> ⚠️ 주의: 사람이 보기 좋은 고품질 변환용이 아니라 **텍스트 분석 파이프라인용**입니다.
> README에도 "meant to be consumed by text analysis tools"라고 명시되어 있습니다.

### 지원 포맷

PDF, Word(docx), PowerPoint(pptx), Excel(xlsx/xls), 이미지(EXIF + OCR), 오디오(EXIF + 음성인식), HTML, CSV/JSON/XML, ZIP(내부 재귀 변환), YouTube URL, EPub, Outlook 메일(.msg), Jupyter 노트북(.ipynb) 등

---

## 2. 폴더 구조 분석

```
markitdown/
├── README.md                     ← 공식 사용법
├── LICENSE                       ← MIT
├── Dockerfile                    ← 도커 빌드
├── SECURITY.md                   ← 보안 취약점 제보 절차
└── packages/
    ├── markitdown/               ⭐ 본체 (라이브러리 + CLI)
    ├── markitdown-mcp/           🔌 MCP 서버 (Claude 등 AI 클라이언트 연결용)
    ├── markitdown-ocr/           👁️ OCR 플러그인 (문서 속 이미지의 글자 인식)
    └── markitdown-sample-plugin/ 🧩 플러그인 제작 예제
```

### 2-1. `packages/markitdown` — 본체

핵심은 `src/markitdown/converters/` 폴더입니다. 포맷별 변환기가 하나씩 들어있습니다.

| 파일 | 담당 |
|---|---|
| `_pdf_converter.py` | PDF |
| `_docx_converter.py` | Word (수식은 `converter_utils/docx/math/`에서 LaTeX로 변환) |
| `_xlsx_converter.py` | Excel (신형) |
| `_pptx_converter.py` | PowerPoint |
| `_image_converter.py` | 이미지 (EXIF 메타데이터 + LLM 이미지 설명) |
| `_audio_converter.py`, `_transcribe_audio.py` | 오디오 → 텍스트 |
| `_youtube_converter.py` | YouTube URL → 자막 |
| `_epub_converter.py` | 전자책 |
| `_zip_converter.py` | ZIP (내부 파일을 재귀적으로 변환) |
| `_outlook_msg_converter.py` | Outlook 메일 |
| `_ipynb_converter.py` | Jupyter 노트북 |
| `_csv_converter.py`, `_plain_text_converter.py` | 텍스트 계열 |
| `_rss_converter.py`, `_wikipedia_converter.py`, `_bing_serp_converter.py` | 웹 특화 |
| `_doc_intel_converter.py`, `_cu_converter.py` | Azure 클라우드 고급 변환 (유료) |

### 2-2. 동작 원리

`_markitdown.py:201-224`에서 변환기들을 **우선순위와 함께 등록**합니다.
파일이 들어오면 각 변환기의 `accepts()`를 호출해 "이 파일 내가 처리 가능?"을 물어보고, 처리 가능한 변환기를 찾아 `convert()`를 실행하는 구조입니다.

덕분에 **플러그인으로 새 포맷을 추가하기 쉽습니다.** (`markitdown-sample-plugin` 참고)

### 2-3. `packages/markitdown-mcp` — MCP 서버

`convert_to_markdown(uri)` 라는 도구 하나를 노출합니다. `http:`, `https:`, `file:`, `data:` URI를 받습니다.

STDIO / Streamable HTTP / SSE 세 가지 전송 방식을 지원합니다.

> ⚠️ README 경고: **로컬 전용**입니다. 인증 기능이 없고, 실행 사용자의 권한으로 아무 파일이나 읽을 수 있습니다.
> localhost 외 인터페이스에 바인딩하지 마세요.

### 2-4. `packages/markitdown-ocr` — OCR 플러그인

PDF/DOCX/PPTX/XLSX **안에 박혀 있는 이미지의 글자**를 LLM Vision(gpt-4o 등)으로 읽어냅니다.
Tesseract 같은 별도 OCR 바이너리 설치가 필요 없습니다.

`llm_client`를 넘기지 않으면 OCR은 조용히 건너뛰고 기본 변환기가 동작합니다.

---

## 3. 설치 방법

### 사전 요구사항

**Python 3.10 이상** (`pyproject.toml`의 `requires-python = ">=3.10"`)

```bash
python3 --version
```

### 가상환경 만들기 (권장)

```bash
cd markitdown

python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
```

프롬프트 앞에 `(.venv)`가 붙으면 성공입니다. 나올 때는 `deactivate`.

### 설치

```bash
# A. 그냥 쓸 경우
pip install 'markitdown[all]'

# B. 이 저장소 소스로 쓸 경우 (편집 모드 - 코드 수정이 즉시 반영됨)
pip install -e 'packages/markitdown[all]'
```

> ⚠️ 따옴표 필수. 없으면 zsh 등에서 `no matches found: [all]` 에러가 납니다.

### 선택적 의존성 목록

본체는 가볍고, 포맷별 라이브러리는 옵션으로 분리되어 있습니다.

| 옵션 | 설명 |
|---|---|
| `[all]` | 전부 설치 (가장 편함) |
| `[pdf]` | PDF (`pdfminer.six`, `pdfplumber`) |
| `[docx]` | Word (`mammoth`, `lxml`) |
| `[pptx]` | PowerPoint (`python-pptx`) |
| `[xlsx]` | Excel 신형 (`pandas`, `openpyxl`) |
| `[xls]` | Excel 구형 (`pandas`, `xlrd`) |
| `[outlook]` | Outlook .msg (`olefile`) |
| `[audio-transcription]` | 음성 인식 (`pydub`, `SpeechRecognition`) |
| `[youtube-transcription]` | YouTube 자막 (`youtube-transcript-api`) |
| `[az-doc-intel]` | Azure Document Intelligence (유료) |
| `[az-content-understanding]` | Azure Content Understanding (유료) |

필요한 것만 고를 수도 있습니다:

```bash
pip install 'markitdown[pdf,docx,pptx]'
```

### 설치 확인

```bash
markitdown --version
```

---

## 4. 사용법 - CLI

### 기본 패턴 4가지

```bash
markitdown 문서.pdf                  # 화면 출력
markitdown 문서.pdf -o 결과.md        # 파일 저장 (-o)
markitdown 문서.pdf > 결과.md         # 파일 저장 (리다이렉트)
cat 문서.pdf | markitdown            # 파이프 입력
```

파일명을 생략하면 stdin에서 읽습니다.

### 실전 예시

```bash
markitdown 이력서.pdf   -o 이력서.md
markitdown 발표자료.pptx -o 발표.md
markitdown 매출.xlsx    -o 매출.md
markitdown 자료모음.zip  -o 전체.md    # ZIP 내부까지 전부
```

### 폴더 일괄 변환

```bash
for f in *.pdf; do
  markitdown "$f" -o "${f%.pdf}.md"
done
```

### 전체 옵션

(`packages/markitdown/src/markitdown/__main__.py` 기준)

| 옵션 | 축약 | 설명 |
|---|---|---|
| `--output <파일>` | `-o` | 결과를 파일로 저장 |
| `--extension <.pdf>` | `-x` | 확장자 힌트 (stdin 입력 시 유용) |
| `--mime-type <타입>` | `-m` | MIME 타입 힌트 |
| `--charset <UTF-8>` | `-c` | 인코딩 지정 (한글 깨질 때) |
| `--use-plugins` | `-p` | 서드파티 플러그인 활성화 |
| `--list-plugins` | | 설치된 플러그인 목록 |
| `--keep-data-uris` | | base64 이미지 등 data URI 유지 (기본은 잘림) |
| `--version` | `-v` | 버전 확인 |
| `--use-docintel` | `-d` | Azure Document Intelligence 사용 |
| `--endpoint <url>` | `-e` | Doc Intelligence 엔드포인트 (`MARKITDOWN_DOCINTEL_ENDPOINT` 환경변수 가능) |
| `--use-cu` | | Azure Content Understanding 사용 |
| `--cu-endpoint <url>` | | CU 엔드포인트 (`MARKITDOWN_CU_ENDPOINT` 환경변수 가능) |
| `--cu-analyzer <id>` | | CU 분석기 지정 |
| `--cu-file-types <목록>` | | CU로 보낼 포맷 제한 (비용 절감) |

### 힌트 옵션이 필요한 경우

파이프로 넘기면 파일명이 전달되지 않아 포맷 추론이 어려울 수 있습니다.

```bash
cat 문서 | markitdown -x .pdf
```

한글이 깨질 때:

```bash
markitdown 파일.csv -c UTF-8
markitdown 파일.csv -c cp949     # 구형 엑셀 CSV
```

---

## 5. 사용법 - Python

### 기본

```python
from markitdown import MarkItDown

md = MarkItDown()
result = md.convert("문서.pdf")

print(result.markdown)   # 변환 결과
print(result.title)      # 문서 제목 (있는 경우)
```

### 파일로 저장

```python
from markitdown import MarkItDown

md = MarkItDown()
result = md.convert("보고서.docx")

with open("보고서.md", "w", encoding="utf-8") as f:
    f.write(result.markdown)
```

### 폴더 일괄 변환

```python
from pathlib import Path
from markitdown import MarkItDown

md = MarkItDown()
out_dir = Path("변환결과")
out_dir.mkdir(exist_ok=True)

for path in Path("내문서").glob("*"):
    if not path.is_file():
        continue
    try:
        result = md.convert(str(path))
        (out_dir / f"{path.stem}.md").write_text(result.markdown, encoding="utf-8")
        print(f"OK  {path.name}")
    except Exception as e:
        print(f"FAIL {path.name} -> {e}")
```

### YouTube 자막

```python
md = MarkItDown()
result = md.convert("https://www.youtube.com/watch?v=영상ID")
print(result.markdown)
```

### 이미지 설명을 LLM으로 생성

```python
from markitdown import MarkItDown
from openai import OpenAI

md = MarkItDown(llm_client=OpenAI(api_key="sk-..."), llm_model="gpt-4o")
result = md.convert("사진.jpg")
```

### convert 계열 함수 구분 (중요)

| 함수 | 범위 |
|---|---|
| `md.convert(...)` | 만능. 로컬 파일 + URL 모두 |
| `md.convert_local("경로")` | 로컬 파일만 |
| `md.convert_stream(f)` | 이미 열린 파일 객체만 |
| `md.convert_uri(uri)` | URI 전용 |

> 🛡️ **보안 원칙:** 신뢰할 수 없는 입력을 다룰 때는 `convert()` 대신 **가장 좁은 함수**를 쓰세요.
> README 상단 경고: MarkItDown은 현재 프로세스 권한으로 I/O를 수행합니다.
> `convert()`는 URL도 받으므로, 사용자 입력을 그대로 넘기면 SSRF나 로컬 파일 유출 위험이 있습니다.

```python
# 권장
result = md.convert_local("/home/user/내파일.pdf")

# 가장 안전
with open("내파일.pdf", "rb") as f:
    result = md.convert_stream(f)
```

---

## 6. Claude 연동 (MCP)

MCP 서버를 붙이면 Claude가 **직접 파일을 열어볼 수 있게** 됩니다.

### 설치 및 실행

```bash
pip install markitdown-mcp

markitdown-mcp                                       # STDIO (기본)
markitdown-mcp --http --host 127.0.0.1 --port 3001   # HTTP/SSE
```

STDIO 모드는 화면에 아무것도 출력하지 않는 것이 정상입니다.

### Claude Desktop 설정

`claude_desktop_config.json` 위치:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "markitdown": {
      "command": "markitdown-mcp"
    }
  }
}
```

저장 후 Claude Desktop을 **완전히 재시작**해야 적용됩니다.

### Claude Code 설정

```bash
claude mcp add markitdown -- markitdown-mcp
```

### 도커로 실행 (README 권장 방식)

```bash
docker build -t markitdown-mcp:latest .
docker run -it --rm -v /home/user/data:/workdir markitdown-mcp:latest
```

```json
{
  "mcpServers": {
    "markitdown": {
      "command": "docker",
      "args": ["run", "--rm", "-i",
               "-v", "/home/user/data:/workdir",
               "markitdown-mcp:latest"]
    }
  }
}
```

도커 쪽이 더 안전합니다. **마운트한 폴더만** 접근 가능하기 때문입니다.

### 디버깅

```bash
npx @modelcontextprotocol/inspector
```

브라우저에서 STDIO 선택 → 명령어 `markitdown-mcp` → Connect → Tools 탭 → `convert_to_markdown` 실행

---

## 7. OCR 플러그인

스캔된 PDF처럼 **글자가 이미지로 되어 있는 문서**에 사용합니다.

```bash
pip install markitdown-ocr
pip install openai
```

### CLI

```bash
markitdown 스캔본.pdf --use-plugins --llm-client openai --llm-model gpt-4o
```

### Python

```python
from markitdown import MarkItDown
from openai import OpenAI

md = MarkItDown(
    enable_plugins=True,
    llm_client=OpenAI(),
    llm_model="gpt-4o",
)
result = md.convert("스캔본.pdf")
```

### 플러그인 확인

```bash
markitdown --list-plugins
```

> 💰 LLM API를 호출하므로 **사용량만큼 과금**됩니다.
> `llm_client`를 주지 않으면 OCR은 건너뛰고 기본 변환기가 동작합니다 (에러 아님).

---

## 8. 자주 만나는 에러

| 증상 | 원인 | 해결 |
|---|---|---|
| `MissingDependencyException` | 해당 포맷 라이브러리 미설치 | `pip install 'markitdown[all]'` |
| `command not found: markitdown` | 가상환경 미활성화 | `source .venv/bin/activate` |
| 한글이 깨짐 | 인코딩 문제 | `-c UTF-8` 또는 `-c cp949` |
| `zsh: no matches found: [all]` | 따옴표 누락 | `'markitdown[all]'` |
| PDF인데 결과가 비어 있음 | 스캔본(이미지) | OCR 플러그인 필요 |
| YouTube 변환 실패 | 자막 없는 영상 | 자막 있는 영상 사용 |

---

## 9. 수익화 검토

### 9-1. 라이선스 확인 결과 — **판매 가능**

`LICENSE` 파일은 **MIT 라이선스**이며, 원문에 명시되어 있습니다:

> "...including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or **sell** copies of the Software..."

즉 상업적 판매가 명시적으로 허용됩니다.

### 지켜야 할 조건

**1. 저작권 고지 유지**
> "The above copyright notice ... shall be included in all copies"

서비스 내 "오픈소스 라이선스" 페이지에 MIT 전문과 `Copyright (c) Microsoft Corporation`을 표기해야 합니다.

**2. Microsoft 상표 사용 금지**

README 399-405줄 상표 조항:
> "Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship."

Microsoft 후원/제작인 것처럼 오인시키면 안 됩니다. **서비스명은 별도로 만드는 것이 안전합니다.**
기술 설명에서 "MarkItDown 기반"이라고 밝히는 것은 무방합니다.

**3. 무보증 조항**
> "THE SOFTWARE IS PROVIDED AS IS, WITHOUT WARRANTY..."

Microsoft는 책임지지 않습니다. 유료 서비스라면 **자체 이용약관에 책임 한계 조항**이 필요합니다.

### 9-2. 냉정한 평가

**"파일을 마크다운으로 변환하는 것" 자체로는 수익화가 어렵습니다.**

- 무료 오픈소스 → 진입장벽 0
- 개발자는 직접 설치하면 그만
- 경쟁 SaaS 다수 존재 (LlamaParse, Unstructured.io, Reducto 등)

> **핵심: MarkItDown은 "재료"이지 "완성품"이 아닙니다.**
> 수익은 변환 자체가 아니라 **변환 앞뒤에 붙이는 워크플로우**에서 나옵니다.

### 9-3. 아이디어 목록

| # | 아이디어 | 난이도 | 수익성 |
|---|---|---|---|
| 1 | 특정 직군 전용 문서 지식창고 SaaS | ⭐⭐⭐ | 💰💰💰💰 |
| 2 | 회의록 자동화 SaaS | ⭐⭐⭐ | 💰💰💰 |
| 3 | YouTube → 콘텐츠 재가공 도구 | ⭐⭐ | 💰💰💰 |
| 4 | RAG 전처리 API | ⭐⭐⭐⭐ | 💰💰💰 |
| 5 | 노코드 자동화 커넥터 (Zapier/Make/n8n) | ⭐⭐ | 💰💰 |
| 6 | HWP 플러그인 오픈소스 배포 | ⭐⭐⭐⭐ | 💰 (간접 효과 큼) |

**1. 특정 직군 전용 문서 지식창고 (추천)**

타겟을 좁히는 것이 핵심입니다. 예: 세무사/노무사 사무실.
고객사 자료(계약서 PDF, 급여대장 엑셀, 스캔 영수증)를 넣으면 자동 변환 → 검색 가능한 지식창고 → 자연어 질의응답.

수익이 나는 이유: 대상이 **개발자가 아니고**, 시간이 곧 비용인 직군이며, 도메인 지식(업계 용어·서식 인식)이 진입장벽이 됩니다.

응용: 병원(논문/가이드라인), 법무(판례/계약서), 건설(시방서), 학원(교재/기출), 중소기업 총무(사규 챗봇)

**2. 회의록 자동화**

`_transcribe_audio.py`가 이미 있습니다. 녹음 → 텍스트 → AI 요약 → 결정사항/할일 추출 → 슬랙·노션 연동.
단, 클로바노트 등 강자가 있으므로 **특정 도메인 용어 특화**로 차별화가 필요합니다.

**3. YouTube 콘텐츠 재가공**

`_youtube_converter.py`로 자막 추출 → 블로그/뉴스레터 자동 생성.
⚠️ 타인 영상 재가공 판매는 저작권 문제가 있으므로 **"내 영상 재활용" 도구**로 포지셔닝해야 안전합니다.

**4. RAG 전처리 API**

변환+청킹+임베딩까지 대행. 설치는 쉽지만 대용량 처리/큐/재시도/스케일링 운영이 귀찮다는 점을 파는 것.
경쟁이 치열하므로 **한국 문서 특화(HWP, 관공서 서식)**가 현실적인 틈새입니다.

**6. HWP 플러그인 (전략적 추천)**

변환기 목록에 **HWP(한글)가 없습니다.** 플러그인 구조가 이미 잘 설계되어 있으므로
`markitdown-hwp` 플러그인을 만들어 오픈소스로 공개하면:

- 국내 개발자 커뮤니티에서 인지도 확보
- 그 과정에서 한국 문서의 실제 문제점 파악 (사업 자산)
- 위 1~4번 사업의 신뢰도·유입 기반

### 9-4. 추천 전략

**"6번 → 1번" 순서**

1. HWP 플러그인으로 기술력 증명 및 이름 알리기 (무료)
2. 그 과정에서 한국 문서의 실제 고통점 파악
3. 그것을 기반으로 직군 특화 SaaS 유료화

한국 시장(HWP, 정부 서식, 한국어 OCR 품질)은 해외 서비스가 접근하기 어려운 영역이라 방어벽이 됩니다.

### 9-5. 사업 시작 전 체크리스트

- [ ] 서비스명 별도 제정 (Microsoft 상표 회피)
- [ ] 오픈소스 라이선스 고지 페이지 (MIT 전문 + MS 저작권)
- [ ] 이용약관에 책임 한계 조항 (MIT 무보증 대응)
- [ ] 개인정보 처리방침 (고객 파일을 다루므로 필수)
- [ ] `convert()` 대신 `convert_local()` / `convert_stream()` 사용
- [ ] 변환 서버를 도커/샌드박스로 격리

---

## 10. PHP 연동 방법

### 10-1. 결론

| 질문 | 답 |
|---|---|
| MarkItDown을 PHP로 재구현 | ❌ 권장하지 않음 |
| PHP로 이걸 활용한 서비스 구축 | ✅ 충분히 가능 |

### 10-2. PHP 재구현이 비효율적인 이유

MarkItDown의 가치는 알고리즘이 아니라 **뒤에 붙은 파이썬 라이브러리 생태계**입니다.

| Python 라이브러리 | 역할 | PHP 대응 |
|---|---|---|
| `pdfminer.six` + `pdfplumber` | PDF 텍스트/표 추출 | 🟡 `smalot/pdfparser` (표 추출 약함) |
| `mammoth` | Word 고품질 변환 | ❌ 없음 |
| `python-pptx` | PowerPoint 파싱 | ❌ 없음 |
| `pandas` + `openpyxl` | Excel | 🟡 `PhpSpreadsheet` |
| **`magika`** | **Google 딥러닝 파일 타입 탐지** | ❌ 이식 사실상 불가 |
| `youtube-transcript-api` | YouTube 자막 | ❌ 없음 |
| `SpeechRecognition` | 음성 인식 | ❌ 없음 |
| `olefile` | Outlook .msg | ❌ 없음 |

특히 `magika`는 확장자가 없거나 틀려도 파일 종류를 판별하는 ML 모델이라 PHP 포팅이 불가능합니다.

### 10-3. 권장 구조

```
   사용자
     ↓
 ┌───────────┐   HTTP    ┌──────────────┐
 │ PHP 서버  │ ────────→ │ Python 변환기 │
 │ (Laravel) │ ←──────── │ (MarkItDown) │
 └───────────┘  마크다운  └──────────────┘
 회원/결제/UI               변환만 담당
 업로드/DB/권한
```

PHP는 웹 서비스(인증·결제·관리자·UI) 구축에 강하고, 변환은 Python에 위임하는 구조입니다.

### 10-4. 방법 1 — PHP에서 CLI 직접 호출

가장 간단합니다. 같은 서버에 `markitdown`을 설치합니다.

```php
<?php
function convertToMarkdown(string $filePath): string
{
    // escapeshellarg 필수 (셸 인젝션 방지)
    $cmd = 'markitdown ' . escapeshellarg($filePath) . ' 2>&1';

    $output = [];
    $exitCode = 0;
    exec($cmd, $output, $exitCode);

    if ($exitCode !== 0) {
        throw new RuntimeException('변환 실패: ' . implode("\n", $output));
    }

    return implode("\n", $output);
}
```

타임아웃까지 처리한 버전:

```php
<?php
function convertSafely(string $filePath, int $timeout = 120): string
{
    $descriptors = [
        1 => ['pipe', 'w'],   // stdout
        2 => ['pipe', 'w'],   // stderr
    ];

    // 배열로 넘기면 셸을 거치지 않아 더 안전
    $process = proc_open(['markitdown', $filePath], $descriptors, $pipes);

    if (!is_resource($process)) {
        throw new RuntimeException('프로세스 실행 실패');
    }

    stream_set_timeout($pipes[1], $timeout);
    $markdown = stream_get_contents($pipes[1]);
    $error    = stream_get_contents($pipes[2]);

    fclose($pipes[1]);
    fclose($pipes[2]);
    $exitCode = proc_close($process);

    if ($exitCode !== 0) {
        throw new RuntimeException("변환 실패: {$error}");
    }

    return $markdown;
}
```

| 장점 | 단점 |
|---|---|
| 구현이 가장 빠름 | PHP+Python을 한 서버에 설치 |
| 구조 단순 | 대용량 파일 시 PHP가 대기 |
| 서버 1대로 완결 | 수평 확장 어려움 |

**적합한 경우:** 사내 도구, MVP, 소규모 트래픽

### 10-5. 방법 2 — Python 마이크로서비스 + PHP REST 호출 (권장)

**Python 쪽 (약 20줄)**

```python
# converter_api.py
from fastapi import FastAPI, UploadFile, HTTPException
from markitdown import MarkItDown
import io

app = FastAPI()
md = MarkItDown()

@app.post("/convert")
async def convert(file: UploadFile):
    try:
        data = await file.read()
        # convert_stream = 디스크에 저장하지 않고 메모리에서 처리 (가장 안전)
        result = md.convert_stream(
            io.BytesIO(data),
            file_extension="." + file.filename.rsplit(".", 1)[-1],
        )
        return {"ok": True, "markdown": result.markdown, "title": result.title}
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
```

```bash
pip install fastapi uvicorn python-multipart 'markitdown[all]'
uvicorn converter_api:app --host 127.0.0.1 --port 8000
```

**PHP 쪽 (Laravel)**

```php
<?php
use Illuminate\Support\Facades\Http;

class ConvertController extends Controller
{
    public function store(Request $request)
    {
        $request->validate([
            'file' => 'required|file|max:51200',   // 50MB
        ]);

        $file = $request->file('file');

        $response = Http::timeout(180)
            ->attach('file', file_get_contents($file->path()), $file->getClientOriginalName())
            ->post('http://127.0.0.1:8000/convert');

        if ($response->failed()) {
            return back()->withErrors('변환에 실패했습니다.');
        }

        return view('result', [
            'markdown' => $response->json('markdown'),
            'title'    => $response->json('title'),
        ]);
    }
}
```

**PHP 쪽 (순수 PHP)**

```php
<?php
$ch = curl_init('http://127.0.0.1:8000/convert');
curl_setopt_array($ch, [
    CURLOPT_POST           => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_TIMEOUT        => 180,
    CURLOPT_POSTFIELDS     => ['file' => new CURLFile('/tmp/문서.pdf')],
]);
$result = json_decode(curl_exec($ch), true);
curl_close($ch);

echo $result['markdown'];
```

| 장점 | 단점 |
|---|---|
| 서버 분리로 수평 확장 용이 | 컨테이너 2개 관리 |
| 변환기 장애가 웹에 전파되지 않음 | 초기 세팅이 조금 더 복잡 |
| 도커 배포가 깔끔 | |

**적합한 경우:** 실제 상용 서비스

### 10-6. 방법 3 — markitdown-mcp HTTP 모드 직접 호출

`packages/markitdown-mcp/src/markitdown_mcp/__main__.py:36-41` 확인 결과:

```python
session_manager = StreamableHTTPSessionManager(
    app=mcp_server,
    event_store=None,
    json_response=True,     # 스트림이 아닌 JSON 응답
    stateless=True,         # 세션 유지 불필요
)
```

`stateless=True`, `json_response=True` 덕분에 PHP에서 단일 POST로 호출하기 좋은 형태입니다.

```bash
markitdown-mcp --http --host 127.0.0.1 --port 3001
```

```php
<?php
$payload = [
    'jsonrpc' => '2.0',
    'id'      => 1,
    'method'  => 'tools/call',
    'params'  => [
        'name'      => 'convert_to_markdown',
        'arguments' => ['uri' => 'file:///var/www/uploads/문서.pdf'],
    ],
];

$ch = curl_init('http://127.0.0.1:3001/mcp');
curl_setopt_array($ch, [
    CURLOPT_POST           => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => [
        'Content-Type: application/json',
        'Accept: application/json, text/event-stream',   // 필수 헤더
    ],
    CURLOPT_POSTFIELDS     => json_encode($payload),
]);
$response = curl_exec($ch);
curl_close($ch);
```

> ⚠️ **미검증 사항:** 위 내용은 소스 코드를 읽고 판단한 것으로, 실제 동작 테스트는 하지 않았습니다.
> MCP 프로토콜 버전에 따라 첫 요청에 `initialize` 핸드셰이크가 필요할 수 있습니다.

| 장점 | 단점 |
|---|---|
| Python 코드를 새로 짤 필요 없음 | JSON-RPC 형식이 번거로움 |
| 기성품 그대로 사용 | 파일 업로드 대신 URI 방식 |
| Claude와 같은 서버 공유 가능 | 실전 검증 필요 |

### 10-7. 도커 컴포즈 예시

```yaml
services:
  web:                        # PHP
    build: ./php
    ports: ["80:80"]
    environment:
      CONVERTER_URL: http://converter:8000
    depends_on: [converter]

  converter:                  # Python
    build: ./converter
    expose: ["8000"]          # ports 아님 - 외부 노출 금지
    user: "nobody:nogroup"
```

`ports` 대신 `expose`를 쓰면 변환 서버는 PHP에서만 접근 가능하고 외부에는 열리지 않습니다.

### 10-8. 방법 비교

| 방법 | 추천도 | 적합한 상황 |
|---|---|---|
| PHP로 재구현 | ❌ | 없음 |
| 방법 1 (CLI 호출) | 🟢 | MVP, 사내 도구 |
| 방법 2 (마이크로서비스) | ⭐⭐⭐ | 상용 서비스 |
| 방법 3 (MCP HTTP) | 🟡 | 실험적, 검증 필요 |

---

## 11. 보안 체크리스트

### 11-1. 셸 인젝션 (최우선)

```php
// 위험 - 절대 금지
exec("markitdown " . $_POST['filename']);
```

파일명에 `문서.pdf; rm -rf /` 같은 값이 들어오면 서버가 손상됩니다.

```php
// 안전
exec("markitdown " . escapeshellarg($path));

// 더 안전 (셸을 거치지 않음)
proc_open(['markitdown', $path], $desc, $pipes);
```

### 11-2. 업로드 파일명 신뢰 금지

```php
$safeName = bin2hex(random_bytes(16)) . '.' . $file->extension();
$path = storage_path('app/uploads/' . $safeName);
```

### 11-3. URI 입력 차단

MCP 서버의 `convert_to_markdown`은 `file:`, `http:` URI를 모두 받습니다.

- `file:///etc/passwd` → 서버 파일 유출
- 내부망 주소 → SSRF

**URI는 반드시 서버가 생성해야 하며, 사용자에게는 파일 업로드만 허용해야 합니다.**

### 11-4. PHP 설정

```ini
upload_max_filesize = 50M
post_max_size = 50M
max_execution_time = 300
memory_limit = 256M
```

### 11-5. 큐 사용

대용량 파일 변환은 오래 걸리므로 동기 처리하면 안 됩니다.

```php
ConvertDocumentJob::dispatch($upload->id);
return response()->json(['status' => 'processing']);
```

### 11-6. 컨테이너 격리

저장소의 `Dockerfile`은 이미 권한 최소화가 되어 있습니다:

```dockerfile
ARG USERID=nobody
ARG GROUPID=nogroup
USER $USERID:$GROUPID
```

**root로 변경하지 마세요.**

---

## 참고 문서

- 이 저장소: https://github.com/bmshin94/markitdown
- 원본 저장소: https://github.com/microsoft/markitdown
- 공식 README: [README.md](README.md)
- 보안 정책: [SECURITY.md](SECURITY.md)
- 라이선스: [LICENSE](LICENSE) (MIT)
- 플러그인 제작 예제: [packages/markitdown-sample-plugin](packages/markitdown-sample-plugin)
- MCP 서버 문서: [packages/markitdown-mcp/README.md](packages/markitdown-mcp/README.md)
- OCR 플러그인 문서: [packages/markitdown-ocr/README.md](packages/markitdown-ocr/README.md)
- MCP 공식 문서: https://modelcontextprotocol.io/
- Azure Content Understanding: https://learn.microsoft.com/azure/ai-services/content-understanding/
