# Impeccable 전수조사 분석 및 활용 정리 (한국어)

> 이 문서는 AI 에이전트(Claude Opus 5, Claude Code)와의 대화를 통해
> 저장소를 전수조사하고 정리한 결과입니다.
> 작성일: 2026-10-07

## 저장소 정보

| 항목 | 값 |
|---|---|
| 내 저장소 (포크) | https://github.com/bmshin94/impeccable |
| 원본 저장소 (upstream) | https://github.com/pbakaus/impeccable |
| 공식 사이트 | https://impeccable.style |
| npm 패키지 | https://www.npmjs.com/package/impeccable |
| VS Code 확장 | https://marketplace.visualstudio.com/items?itemName=renaissance-geek.impeccable |
| 제작자 | Paul Bakaus |
| 라이선스 | Apache-2.0 (상업적 이용, 수정, 재배포 자유) |
| npm 버전 | 4.1.0 |
| 스킬 / 플러그인 버전 | 4.3.1 |
| 엔진 버전 | 0.1.5 |
| 크롬 확장 버전 | 1.4.0 |
| 규모 | 파일 3,140개 / Rust 크레이트 16개 / 지원 하네스 17개 |
| 포크 차이 | main 기준 커밋 1개 추가 (CLAUDE.md 프로젝트 가이드). 코드 변경 없음 |

---

## 1. 이게 뭐하는 프로젝트인가

### 한 줄 정의

AI 코딩 도구가 만든 UI에서 "AI 티(AI slop)"를 제거하고 프로급 디자인 품질을 보장하는
오픈소스 디자인 품질 보증 시스템.

### 문제 정의 (README 인용)

> 모든 모델이 똑같은 SaaS 템플릿으로 학습됐다. 가이던스 없이 돌리면 모든 프로젝트에서
> 똑같은 몇 가지 흔적이 나온다: 전부 Inter 폰트, 보라에서 파랑으로 넘어가는 그라데이션,
> 카드 안에 카드, 컬러 배경 위 회색 글씨, 모든 제목 위에 둥근 사각형 아이콘 타일.

Anthropic 공식 `frontend-design` 스킬에서 출발해, 결정론적 검사기와 라이브 브라우저 반복,
24개 커맨드 체계를 얹은 것이 Impeccable입니다.

### 아키텍처: 3개 레이어

```
① 스킬 레이어 (LLM이 읽는 마크다운)
   skill/SKILL.src.md + skill/reference/*.md (36개)
   24개 커맨드. 판단이 필요한 영역 (미학, 위계, 감성)

② 결정론적 디텍터 (Rust, LLM 0회, API키 0개)
   crates/foundation(룰 레지스트리) + crates/core(체크 로직)
   61개 룰. CLI / 에디터 훅 / 크롬 확장 / WASM 공용
   기계가 확실히 판정 가능한 영역 (대비비, 줄길이, 폰트)

③ 라이브 모드 (브라우저 오버레이 + dev server HMR)
   crates/live + skill/scripts/live-browser.js (547KB)
   브라우저에서 요소 클릭, AI가 변형안 N개 생성, 슬라이더로 비교, 선택한 것만 소스 반영
```

핵심 설계 사상: **LLM이 잘하는 것과 코드가 잘하는 것을 섞지 않았습니다.**

### 폴더 구조

| 폴더 | 역할 |
|---|---|
| `skill/` | **제품의 진짜 본체.** 사람이 직접 작성하는 마크다운 지침서 |
| `skill/SKILL.src.md` | 프론트매터 + 공통 디자인 법칙 + 커맨드 라우터 표 (12KB) |
| `skill/reference/` | 36개 커맨드별 플레이북. 네이티브 변종(ios/android) 포함 |
| `skill/scripts/` | 런처(`impeccable`, `.cmd`), `command-metadata.json`, 라이브 오버레이 JS |
| `skill/agents/` | 서브에이전트 4개 (documenter, finish-reviewer, asset-producer, manual-edit-applier) |
| `crates/` | Rust 워크스페이스 16개. 엔진 바이너리 `impeccable` |
| `cli/` | npm 패키지. cli.js는 바이너리를 찾아 exec하는 얇은 shim |
| `extension/` | 크롬 확장 MV3. 오프스크린 문서에서 WASM으로 룰 실행 |
| `plugin/`, `cursor-plugin/`, `vscode/` | 플러그인 / 확장 패키징 |
| `.claude/ .cursor/ .gemini/ .codex/ ...` | 17개 하네스별 **빌드 산출물**. 손대지 말 것 |
| `scripts/` | Bun 빌드 시스템. 플레이스홀더 치환 + 검증 게이트 |
| `tests/` | oracle 골든 / 프레임워크 픽스처 30개 / live-e2e / LLM 백드 행동 테스트 |
| `docs/` | CLI-CONTRACT.md (318KB), ENGINE.md, STYLE.md 등 설계 문서 |

### 주요 Rust 크레이트

| 크레이트 | 역할 |
|---|---|
| `foundation` | 룰 레지스트리, 색 처리, `Dom` trait, **룰 팩 API(`RulePack`)** |
| `core` | 61개 체크 로직 본체, 브라우저 룰 어댑터, 시각 대비 판정 |
| `html` | HTML 파싱 + CSS 캐스케이드 (정적 분석) |
| `browser` | CDP 연결, 렌더링된 DOM 스냅샷 |
| `detect` | 파일 워커 + 출력 |
| `live` | 라이브 모드 서버 (HTTP 롱폴링, 변형안 퍼블리시, 소스 반영) |
| `context` | `impeccable context` verb, 이미지 생성, 질문 페이지 |
| `hook` | 에디터 편집 시 자동 스캔 |
| `wasm` | **같은 소스를 WebAssembly로.** 확장 / 라이브 오버레이 / 웹 공용 |
| `comp`, `comp-verbs` | comp-first 빌드 경로 |
| `skills` | install / update / link 설치 로직 |
| `cli` | 바이너리 엔트리. 28개 verb 라우팅 |
| `bundle`, `xtask` | 브라우저 번들 빌드 |

---

## 2. 24개 커맨드

| 그룹 | 커맨드 | 하는 일 |
|---|---|---|
| Build | `init` | 1회 세팅. 프로젝트 조사 후 부족한 것만 질문, `PRODUCT.md` 작성 |
| Build | `document` | 기존 코드에서 `DESIGN.md`(디자인 시스템) 역추출 |
| Build | `extract` | 재사용 컴포넌트와 토큰을 디자인 시스템으로 승격 |
| Build | `shape` | 코드 쓰기 전 UX/UI 설계 |
| Build | `craft` | (deprecated) 일반 신규 작업의 별칭 |
| Evaluate | `critique` | UX 디자인 리뷰: 위계, 명료성, 감성적 울림 |
| Evaluate | `audit` | 기술 품질: 접근성(a11y), 성능, 반응형 |
| Refine | `polish` | 출하 전 마무리 + 디자인 시스템 정합성 |
| Refine | `bolder` | 심심한 디자인 증폭 |
| Refine | `quieter` | 과한 디자인 진정 |
| Refine | `distill` | 본질만 남기고 제거 |
| Refine | `harden` | 에러 처리, i18n, 텍스트 넘침, 엣지 케이스 |
| Refine | `onboard` | 첫 실행 흐름, 빈 상태, 활성화 경로 |
| Enhance | `animate` | 목적 있는 모션 |
| Enhance | `colorize` | 전략적 색 투입 |
| Enhance | `typeset` | 폰트, 위계, 크기 |
| Enhance | `layout` | 간격, 리듬, 시각 위계 |
| Enhance | `delight` | 기억에 남는 터치 |
| Enhance | `overdrive` | 기술적으로 극단적인 효과 |
| Fix | `clarify` | UX 문구, 레이블, 에러 메시지 |
| Fix | `adapt` | 디바이스, 화면 크기 적응 |
| Fix | `optimize` | UI 성능 진단 및 수정 |
| Iterate | `live` | 브라우저에서 요소 골라 변형안 비교 |
| Iterate | `generate` | 지정 요소의 변형 N개 생성 (수동 선택 없이) |

유틸리티 (디자인 메뉴에서 의도적으로 제외): `pin`(단축 커맨드 생성), `hooks`, `doctor`

> 참고: README와 `command-metadata.json`은 24, `PRODUCT.md`는 23으로 적혀 있습니다.
> `craft`가 deprecated 별칭이라 세는 방식에 따라 갈립니다.

---

## 3. 61개 디텍터 룰

### AI slop 계열 32개 (AI가 만들었다는 티)

`side-tab` `border-accent-on-rounded` `overused-font` `flat-type-hierarchy` `gradient-text`
`ai-color-palette` `cream-palette` `nested-cards` `monotonous-spacing` `bounce-easing`
`pulsing-dot` `blinking-cursor` `shape-assembled-illustration` `dark-glow` `radial-halo`
`radial-spotlight-glow` `marquee` `icon-tile-stack` `italic-serif-display` `hero-eyebrow-chip`
`kicker-above-heading` `numbered-section-labels` `em-dash-overuse` `marketing-buzzword`
`aphoristic-cadence` `oversized-h1` `extreme-negative-tracking` `gpt-thin-border-wide-shadow`
`repeating-stripes-gradient` `codex-grid-background` `theater-slop-phrase` `image-hover-transform`

### 일반 품질 29개

`broken-image` `script-error` `content-hidden-at-rest` `edge-flush-cards` `text-occlusion`
`first-viewport-column-overflow` `gray-on-color` `low-contrast` `layout-transition`
`line-length` `cramped-padding` `body-text-viewport-edge` `tight-leading` `skipped-heading`
`heading-rhythm` `justified-text` `tiny-text` `undersized-ui-text` `all-caps-body`
`wide-tracking` `text-overflow` `repeated-container-text` `clipped-overflow-container`
`organic-clip-path` `buried-raster`
+ 디자인 시스템 이탈 4종: `design-system-font` `design-system-color` `design-system-radius`
`design-system-font-size`

마지막 4개가 실무에서 특히 유용합니다. `DESIGN.md`를 한 번 정의해두면
"우리 디자인 시스템 밖의 값"을 기계적으로 잡아냅니다.

---

## 4. 모드와 플랫폼 (직교하는 두 축)

### 모드 (방문자의 성공이 무엇인가). 프로젝트 단위가 아니라 화면 단위

| 모드 | 방문자가 하는 일 | 예 |
|---|---|---|
| Persuade | 결정하고 행동 (디자인이 제품) | 랜딩, 마케팅, 캠페인, 가격 |
| Operate | 과업 완수 | 앱 UI, 대시보드, 에디터, 관리자, 설정 |
| Read | 이해 | 문서, 아티클, 가이드, 체인지로그 |
| Experience | 작업물 안에 들어감 | 포트폴리오, 갤러리, 쇼케이스 |

핵심: 개발툴의 랜딩 페이지는 Persuade. 패션 하우스의 문서는 Read.
제품이 아니라 화면이 모드를 결정합니다.
모드는 `PRODUCT.md`에 저장하지 않고 그 화면의 brief(`.impeccable/surfaces/`)에만 저장합니다.

### 플랫폼 (배포 타깃)

`web`(기본) / `ios`(Apple HIG) / `android`(Material 3) / `adaptive`(둘 다, Flutter/RN/KMP)

`PRODUCT.md`의 `## Platform`에서 읽고, 네이티브면 `impeccable context`가 해당 레퍼런스를
출력에 인라인해서 추가 읽기 없이 전달합니다.
라이브 모드와 `detect`, 디자인 훅은 **웹 전용**입니다.

### 세 가지 핵심 파일

| 파일 | 역할 | 누가 쓰나 |
|---|---|---|
| `PRODUCT.md` | 신분증. 사용자, 목적, 운영 맥락, 제약, 브랜드 목소리, 플랫폼 | `init`이 1회 |
| `DESIGN.md` | 디자인 설계도. 코드에서 역추출한 폰트/색/간격/반경 토큰 | `document` |
| `.impeccable/surfaces/*.md` | 화면별 작업 지시서. 모드, 비주얼 방향, 방향 계약 | 화면 작업마다 |

---

## 5. 설치 및 사용법

### 사용자 설치 (추천)

```bash
# 프로젝트 루트에서
npx impeccable install

# 스크립트에서 질문 건너뛰기
npx impeccable install --providers=claude,cursor --scope=project --no-hooks

# AI 세션에서
/impeccable init

# 이후 사용
/impeccable critique 랜딩페이지
/impeccable audit
/impeccable polish 체크아웃폼
/impeccable                      # 커맨드 메뉴
/impeccable pin audit            # /audit 단축키 생성

# 업데이트
npx impeccable update
```

### 다른 설치 경로

```bash
# Claude Code 플러그인 (가장 간편)
/plugin marketplace add pbakaus/impeccable

# Grok Build
grok plugin install pbakaus/impeccable#plugin --trust

# VS Code (Copilot)
code --install-extension renaissance-geek.impeccable

# Git 서브모듈 (팀 버전 고정)
git submodule add https://github.com/pbakaus/impeccable .impeccable
npx impeccable link --source=.impeccable --providers=claude,cursor
```

### AI 없이 CLI만 쓰기

```bash
npx impeccable detect src/                    # 디렉터리 스캔
npx impeccable detect index.html              # 파일 스캔
npx impeccable detect https://example.com     # URL (설치된 Chrome/Edge 사용)
npx impeccable detect --json .                # CI용 JSON
npx impeccable detect --no-config src/        # 프로젝트 설정 무시

npx impeccable ignores list
npx impeccable ignores add-file "src/legacy/**"
npx impeccable ignores add-value overused-font Inter --reason "브랜드 폰트"
```

종료 코드: `0`=발견 없음, `2`=발견 있음, `1`=스캔 실패.
사람이 읽는 출력은 stderr, JSON은 stdout.

파일 단위 예외: `<!-- impeccable-disable overused-font: 브랜드 지정 -->`
(어떤 주석 문법이든 동작. `-line`, `-next-line` 변종 있음)

### 스캔 가능 확장자

`.html .htm .css .scss .sass .less .jsx .tsx .js .ts .vue .svelte .astro .blade.php`

주의: `.blade.php`는 되지만 순수 `.php`는 스캔 대상이 아닙니다.
(`crates/detect/src/file_system.rs` 테스트에 명시)

### 개발자 빌드

```bash
# 사전 요구: Node 22.18+, Bun, Rust stable
bun install
bun run fetch:engine                 # 또는 cargo build --release -p impeccable

bun run build                        # dist/ 생성 (루트 하네스 동기화 없음)
bun run build:release                # 커밋되는 하네스 폴더까지 동기화
bun run test                         # 기본 스위트
cargo test --workspace               # Rust 유닛/통합

IMPECCABLE_BIN="$PWD/target/release/impeccable" bun run test
```

중요: `.claude/` `.cursor/` 같은 루트 폴더는 빌드 산출물입니다.
`skill/`을 고치고 빌드하세요. 피처 PR에 생성물을 넣지 않는 것이 이 저장소 규칙입니다.

---

## 6. 플러그인인가, 스킬인가, MCP인가

**본질은 "스킬 + 독립 CLI 엔진"입니다. 플러그인은 포장 방식 중 하나. MCP는 전혀 아닙니다.**

저장소 전체에 `modelcontextprotocol`, `mcpServers`, MCP 서버 구현이 하나도 없습니다.
(유일한 "mcp" 문자열은 플러그인 매니페스트 검증 테스트에서
`mcpServers` 키가 허용되지 않음을 확인하는 코드)

| 개념 | 정의 | Impeccable은? |
|---|---|---|
| 스킬 | AI가 읽는 마크다운 지침서 패키지 | **이게 본질** |
| 플러그인 | 하네스별 배포 포맷 | 포장 방식 (Claude Code/Grok/Cursor/VS Code) |
| MCP | AI가 외부 도구에 연결하는 프로토콜. 서버가 떠서 툴 노출 | **아님** |
| CLI 도구 | 터미널 프로그램 | 이것도 맞음 (`npx impeccable detect`) |

### 왜 MCP가 아닌가

MCP는 런타임 통신 프로토콜입니다. Impeccable이 하는 일은 대부분 "모델이 읽는 지침"이고,
지침 전달에는 서버가 필요 없습니다. 코드 실행이 필요할 때는 `allowed-tools`에 선언된
Bash 호출을 씁니다:

```yaml
allowed-tools:
  - Bash(npx impeccable *)
  - Bash({{scripts_path}}/impeccable *)
```

MCP 서버 대신 바이너리 직접 실행을 선택한 이유: 의존성이 적고(Node조차 불필요),
하네스 17개를 동시 지원하기 쉽습니다.

라이브 모드는 로컬 HTTP 서버를 띄우지만 이건 MCP가 아니라
브라우저와 에이전트 사이의 자체 롱폴링 프로토콜입니다.

---

## 7. API 토큰이 필요한가

**기본 사용에는 전혀 필요 없습니다.**

### API 키 0개로 동작하는 것

| 기능 | 이유 |
|---|---|
| `npx impeccable detect` | 순수 Rust 결정론적 코드. LLM 호출 0회 |
| 크롬 확장 | 같은 룰을 WASM으로 브라우저에서 실행 |
| 에디터 훅 | 같은 디텍터를 편집 시 호출 |
| **24개 스킬 커맨드 전부** | 이미 쓰는 하네스의 모델을 그대로 사용. 별도 키 요구 없음 |
| 라이브 모드 변형 생성 | 하네스 모델이 같은 스레드에서 생성 |

핵심: Impeccable은 자체적으로 LLM을 호출하지 않습니다.

### 선택적으로 키가 필요한 3곳

1. `generate-image` verb (comp-first 경로): `OPENAI_API_KEY`.
   없으면 "하네스 내장 이미지 도구를 쓰라"는 안내만 출력하고 넘어갑니다. 차단 아님.
   `buildPath: "code"`를 쓰면 애초에 필요 없습니다.
2. 라이브 모드 copy-edit 에이전트: `ANTHROPIC_API_KEY`가 **대안** 경로로 안내됩니다.
3. 개발자용 테스트 스위트 (`test:skill-behavior` 등). 키 없으면 깔끔하게 스킵됩니다.
   기본 `bun run test`는 키 0개로 동작합니다.

### 네트워크

| 상황 | 네트워크 |
|---|---|
| 첫 실행 시 엔진 바이너리 다운로드 | 1회. GitHub Releases → `~/.impeccable/bin/`. 체크섬 검증 |
| 그 이후 detect / hook | 완전 오프라인 |
| `concept-seed` | `IMPECCABLE_CATALOG_DIR` → roll API → 폴백. 키 불필요 |
| 텔레메트리 | `--chosen` 선택 핑만. `DO_NOT_TRACK` / `IMPECCABLE_NO_TELEMETRY` 준수 |

---

## 8. AI 에이전트 구축에 도움이 되는가

**매우 큰 도움이 됩니다.**

### 층위 A: 내 에이전트의 출력 품질을 올리는 부품

| 활용 | 방법 | 난이도 |
|---|---|---|
| 디자인 지침 주입 | `SKILL.src.md` + 필요한 `reference/*.md`를 시스템 프롬프트에 인라인 | 쉬움 |
| **자동 검증 루프** | 생성 → `detect --json` → 발견사항 피드백 → 재생성 | 쉬움, 가장 실용적 |
| 브라우저/웹 내장 | `crates/wasm`의 `detect_text_json`, `detect_html_source_json` | 중 |
| 내 도메인 룰 추가 | `RulePack` trait 구현 + `install(&PACK)`. 포크 불필요 | 중상 |

검증 루프 패턴:

```
에이전트가 UI 생성
  → impeccable detect --json  (LLM 0회, 밀리초, 비용 0)
  → exit code 2면 발견사항 피드백 후 재생성 (최대 N회)
  → 아니면 통과
```

LLM-as-judge로 같은 걸 하면 호출마다 돈과 지연이 들고 판정이 흔들립니다.
결정론적 검사기는 무료, 즉시, 재현 가능합니다.

### 층위 B: 에이전트/스킬 설계의 교과서

1. **판단 레이어와 결정론 레이어의 분리.**
   대비비 4.5:1 미달은 코드가 판정, "설득력이 있는가"는 LLM이 판정.
2. **라우터 패턴으로 컨텍스트 절약.**
   SKILL.md(12KB)는 항상 로드, 해당 `reference/*.md`만 당겨옴.
   v4에서 도메인별 레퍼런스를 전부 삭제하고 커맨드 레퍼런스로 흡수.
3. **"하지 말 것"을 명시적으로 쓰기.**
   각 규칙에 `<!-- rule:skill-brief-wins -->` 같은 ID가 붙어 테스트가 검증합니다.
4. **과잉 자기검증 방지를 명시적 규칙으로.**
   "Verify in bounded passes, not a loop. Open-ended self-QA burns the user's money."
   에이전트에게 "검증해라"만 하면 무한 루프에 빠집니다. 상한선을 써야 합니다.
5. **심리적 역효과를 고려한 지침 배치.**
   접근성을 디자인 시점에 상기시키면 모델이 과도하게 보수적으로 변해
   결과물이 빈약해집니다. 그래서 `audit.md`로 분리했습니다.
   deprecated 필드도 **이유를 적어야** 모델이 버립니다. 안 적으면 "혹시 몰라서" 보존합니다.
6. **3단 테스트 전략.**
   oracle 골든(비용 0) / 프레임워크 픽스처 30개(비용 0) /
   LLM 백드 행동 테스트($0.5~1.5). 세 번째는 모델의 자유 서술이 아니라
   **도구 호출 트레이스**를 검증합니다. "The trace is the source of truth."
   그리고 "Claude만 쓰지 말 것. 가장 유용한 발견은 모델 패밀리 간 차이에서 나온다."
7. **멀티 하네스 배포 파이프라인.** 하나의 소스에서 17개 툴용 산출물을 플레이스홀더 치환으로.
8. **아티팩트 노후화 관리.** 툴 버전 드리프트 / 스키마 드리프트 / 진실 드리프트로 3분할.
   발견사항을 `{ id, artifact, path, severity, summary, fix }` 데이터로 통일하고,
   severity는 "얼마나 나쁜가"가 아니라 **"무엇을 해야 하는가"**(`auto`/`mention`/`route`).
9. **Tier 1 / Tier 2 성능 계약.**
   부팅 시(Tier 1)에는 디렉터리 순회와 git 금지.
   "여기에 비싼 검사를 넣으면 모든 프로젝트의 모든 세션에 세금을 매긴다."
10. **프로세스 누출 방지 (실전 교훈, 이슈 #717).**
    테스트 서버를 죽일 때 프로세스명/포트/경로가 아니라 **환경변수 마커로만** 매칭합니다.
    다른 체크아웃이나 사용자 세션의 서버를 절대 건드리지 않기 위해서입니다.
    마커 값은 공백을 포함할 수 없게 강제합니다. `ps -E`가 환경을 공백 구분 한 줄로
    평탄화하기 때문에 공백이 들어가면 한 체크아웃의 정리가 다른 체크아웃에 도달할 수 있습니다.

---

## 9. React나 PHP로 만들 수 있는가

### 해석 A: React/PHP 프로젝트에 Impeccable을 쓸 수 있나

**React: 완벽 지원.** `tests/framework-fixtures/`의 30개 픽스처 대부분이 React입니다.

```
vite-react, vite8-react-plain, vite8-react-base-path, vite8-react-css-modules,
vite8-react-emotion, vite8-react-styled-components, vite8-react-insert,
vite8-react-modal, vite8-react-pricing-cards, vite8-react-radix-dialog,
vite8-react-router-spa, vite8-react-mapped-list, vite8-react-csp-meta,
nextjs-app, nextjs-app-router, nextjs-turborepo, nextjs-inline-csp,
nextjs-proxy-csp, tanstack-router-vite, tanstack-start,
monorepo-nested-vite, vite8-https, astro, sveltekit, nuxt-csp ...
```

CSS-in-JS도 파싱합니다 (Emotion / styled-components 블록 추출).
React Native는 `platform: ios/android/adaptive`로 지원되며,
이 경우 라이브 모드와 훅 스캔은 자동 비활성화됩니다.

**PHP: 부분 지원.**

| PHP 상황 | 지원 |
|---|---|
| Laravel Blade (`.blade.php`) | 정적 스캔 가능 |
| 순수 `.php` | 스캔 대상 아님 |
| PHP 결과물을 URL로 검사 | **완전히 가능.** `npx impeccable detect http://localhost:8000` |
| 분리된 `.css` 파일 | 가능 |
| 라이브 모드 | 불가 (HMR dev server 필요) |
| 스킬 커맨드 | **가능. 언어 무관** (마크다운 지침이라 AI가 PHP 코드도 고침) |

실무 권장: 스킬 커맨드는 그냥 쓰고, 검사는 `.blade.php` + `.css` 정적 스캔과
**로컬 URL 스캔**을 병용. URL 스캔은 렌더링된 DOM과 계산된 레이아웃을 보므로
백엔드 언어와 무관하게 가장 정확합니다.

기회: `.php` 확장자 추가는 작은 변경입니다 (`SCANNABLE_EXTENSIONS` + 템플릿 추출 처리).
단 이 저장소는 issue-first 정책이므로 PR 전에 이슈를 먼저 열어야 합니다.

### 해석 B: Impeccable 같은 걸 React/PHP로 직접 만들 수 있나

| 레이어 | 재현 난이도 | 평가 |
|---|---|---|
| 스킬 마크다운 | 쉬움 | 언어 무관. Apache-2.0으로 포크/번역/수정 자유 |
| 디텍터 61룰 | 어려움 | JS는 현실적(원래 JS였음). PHP는 권장하지 않음 |
| 라이브 모드 | 매우 어려움 | 브라우저 오버레이 + HMR + 소스 역반영 |

JS/Node 재현 시: parse5/cheerio(HTML), postcss(CSS), Playwright(computed style),
culori(OKLCH). CSS 캐스케이드 직접 구현이 가장 어렵습니다.

PHP 재현 시: DOMDocument는 되지만 캐스케이드/computed style과
헤드리스 브라우저 제어가 약합니다.

**핵심 통찰:** 정적 소스를 보는 것과 렌더링된 결과를 보는 것은 난이도가 다릅니다.
"대비비 4.5:1"을 판정하려면 실제 적용된 전경색과 배경색을 알아야 하고,
그건 캐스케이드 전체를 해석해야 나옵니다. Impeccable이 CDP를 쓰는 이유입니다.

**현실적 전략: 복제하지 말고 위에 얹으세요.**

```
내가 만드는 것 (React/PHP)
  웹 대시보드 (검사 결과 시각화)
  팀 리포트, 추이 그래프, PR 코멘트
  한글 타이포그래피 룰 (차별점)
  커스텀 룰 설정 UI
        |
        v  호출
  impeccable detect --json   (그냥 씀. Apache-2.0)
        또는
  crates/wasm 의 detect_text_json  (브라우저에서 직접)
```

`--json`을 소비하는 React 대시보드는 며칠이면 만듭니다.
61룰 재구현은 몇 달이고, 재구현해도 upstream 룰 추가 때마다 뒤처집니다.

룰만 추가하려면 `RulePack` trait (Rust, 훅 3개).
`crates/core/tests/rule_pack.rs`, `crates/html/tests/rule_pack.rs`,
`crates/detect/tests/rule_pack.rs`가 그대로 템플릿입니다.

---

## 10. 유튜브 강의 제작 가능성

### 법적 측면: 완전히 가능

| 항목 | 상태 |
|---|---|
| 라이선스 | Apache-2.0 |
| 상업적 이용 / 유료 강의 판매 | 허용 |
| 코드 화면 노출, 수정, 포크 | 허용 |
| 의무 | 저작권 고지 + 라이선스 사본 유지, 변경 사항 표시. 영상은 설명란 표기로 충분 |
| 주의 1 | "Impeccable 공식 강의"처럼 공식인 것처럼 오해시키지 말 것 |
| 주의 2 | `NOTICE.md`: `ios.md`/`android.md`는 MIT 라이선스 `ehmo/platform-design-skills` 파생. 재배포 시 고지 유지 |

### 콘텐츠로서의 강점

- 시각적으로 강력 (Before/After가 즉시 이해됨)
- 타이밍이 좋음 (AI 코딩 사용자 폭증 + "결과물이 다 똑같다"는 보편적 불만)
- 한국어 콘텐츠가 거의 공백
- "61개 룰"처럼 숫자가 명확해서 제목과 썸네일이 잘 뽑힘
- 입문(npx 한 줄)부터 고급(Rust 룰 팩)까지 시리즈 확장 가능

### 약점과 대응

| 약점 | 대응 |
|---|---|
| 프로젝트가 빠르게 변함 | 버전 명시, 핵심 개념 중심 구성 |
| 설치 경로가 17개로 복잡 | Claude Code 하나로 고정, 나머지는 한 줄 처리 |
| 시청자가 AI 툴을 안 쓸 수 있음 | **첫 편을 `detect`로 시작.** AI 툴 없이 누구나 따라할 수 있음 |
| 디자인 지식 필요 | 룰 하나씩 Before/After로 설명하면 오히려 디자인 교육 콘텐츠가 됨 |

### 추천 커리큘럼

**시즌 1: 문제 인식과 즉시 효과**

| # | 제목 안 | 내용 |
|---|---|---|
| 1 | AI가 만든 웹사이트는 왜 다 똑같이 생겼을까 | 문제 정의. AI 생성 사이트 5개 공통 패턴 |
| 2 | 내 사이트 디자인 점수 1분 만에 확인하기 | `detect` 하나로 끝. AI 툴 불필요. 유명 사이트 비교 |
| 3 | 61개 룰 전부 뜯어보기: AI slop 32개 | 룰별 Before/After + 디자인 원리 |
| 4 | 61개 룰 전부 뜯어보기: 품질 29개 | 대비비, 줄 길이, 헤딩, 터치 타깃. 접근성 교육 겸함 |

**시즌 2: AI와 함께 쓰기**

| # | 제목 안 |
|---|---|
| 5 | 설치부터 첫 커맨드까지 (PRODUCT.md가 뭔지) |
| 6 | critique: AI에게 디자인 디렉터 역할 시키기 |
| 7 | polish / bolder / quieter: 같은 화면 3가지 방향 |
| 8 | audit: 접근성, 성능, 반응형 한 번에 |
| 9 | DESIGN.md 만들고 디자인 시스템 지키기 (가장 실무적) |
| 10 | live 모드: 브라우저에서 실시간 비교 (가장 화려함) |

**시즌 3: 고급, 개발자**

| # | 제목 안 |
|---|---|
| 11 | 훅으로 편집할 때마다 자동 검사 (Cursor는 사전 차단) |
| 12 | CI에 디자인 품질 게이트 걸기 (GitHub Actions + exit code 2) |
| 13 | Rust로 내 룰 추가하기 (한글 타이포 룰 실습) |
| 14 | 소스 해부: 프로덕션 AI 스킬 설계법 (에이전트 개발자 타깃) |
| 15 | 한국 웹사이트 10개 실전 분석 (조회수 최상위 예상) |

### 제작 팁

- 첫 편은 2번(유명 사이트 검사)으로 올리는 것도 좋습니다. 진입 장벽 0, 바이럴 요소
- 데모 소재: `demos/landing-demo/`에 before/after 세트가 이미 있습니다 (`classic/`이 before)
- 윤리: 대기업/공공 위주, 조롱 아닌 교육 톤, "자동 검사기 결과이며 디자인 전체 평가가
  아니다"는 면책 필수. README에도 적혀 있습니다:
  "깨끗한 디텍터 결과는 증거일 뿐 증명이 아니다"
- 차별화: 한국어 + 한글 타이포그래피 관점 + 실제 한국 사이트 분석.
  이 조합은 현재 세계적으로 없습니다

---

## 11. 수익화 아이디어

### 전제: 시장 구조

원저작자가 이미 수익화 중입니다. `CLAUDE.md`에 비즈니스 모델이 명시돼 있습니다:

> v4부터 이 저장소는 오픈소스 **제품 레이어**만 보유합니다. 서비스 쪽은 전부
> 비공개 저장소 `pbakaus/impeccable-site`에 있습니다: 사이트, 리뷰 랩,
> **concept/composition 카탈로그**, 월드카드 이미지 파이프라인, Cloudflare Functions.
>
> "카탈로그 데이터 파일을 이 저장소에 다시 넣지 말 것. **카탈로그가 유료 서비스의 모트다.**"

즉 **코드는 공개, 큐레이션 데이터는 비공개 = 유료**입니다.

### 그래서 전략은 정면 경쟁이 아니라 보완/지역화/전문화

| 전략 | 평가 |
|---|---|
| 똑같은 걸 만들어 경쟁 | 승산 낮음. 모트가 데이터 |
| **지역화** (한국어, 한글 타이포) | 완전 공백 |
| **수직 전문화** (산업별 룰 팩) | 룰 팩 API가 공식 지원 |
| **교육/서비스** (강의, 컨설팅) | 코드 경쟁 아님. 라이선스 자유 |
| **상위 레이어** (대시보드, CI SaaS) | `--json` 소비. 재구현 불필요 |

### 결정적 공백: 한글(CJK) 타이포그래피 룰이 0개

61룰을 전수 확인한 결과, CJK 전용 룰이 **하나도 없습니다.**
`line-length`, `tight-leading`, `extreme-negative-tracking`, `wide-tracking`, `tiny-text`
전부 라틴 문자 기준입니다. 그런데:

| 항목 | 라틴 | 한글 |
|---|---|---|
| 적정 줄 길이 | 45~75자 | 글자당 정보량이 달라 기준이 다름 |
| 적정 행간 | 1.5 전후 | **더 넓어야 함 (1.7~1.8).** 받침 때문에 글자 높이가 들쭉날쭉 |
| letter-spacing | 음수 트래킹이 디스플레이에 유효 | **음수 트래킹이 거의 항상 가독성 파괴** |
| 최소 본문 크기 | 16px | **더 커야 함.** 획이 복잡해 작으면 뭉개짐 |
| 폰트 폴백 | - | **한글 웹폰트 미지정 시 OS별로 완전히 다른 폰트** |
| 금칙 처리 | 하이픈 | **`word-break: keep-all` 미적용 시 어색하게 끊김** |
| 과용 폰트 | Inter, Arial | **Noto Sans KR / 본고딕 단독 사용이 정확히 같은 문제** |

### 아이디어 10개

#### 1위. 한국어 교육 콘텐츠 (유튜브 → 유료 강의)

| 항목 | 내용 |
|---|---|
| 수익 모델 | 유튜브 광고 → 인프런/클래스101 강의 → 멤버십 |
| 초기 비용 | 거의 0 |
| 난이도 | 낮음 |
| 왜 1순위 | 리스크 최저 + **다른 모든 아이디어의 마케팅 채널** |

실행: 무료 유튜브 15편(위 커리큘럼) → 인프런 유료 강의(10~15시간) →
강의 특전으로 한글 룰 팩 번들해 상위 전환.

리스크: 한국 AI 코딩 사용자 모집단이 아직 작음.
대응: "AI 안 써도 쓸 수 있는 웹 디자인 린터"로도 포지셔닝.

#### 2위. 한글 타이포그래피 룰 팩 (제품)

| 항목 | 내용 |
|---|---|
| 수익 모델 | 무료 5룰 + 유료 Pro 20~30룰 / 팀 라이선스 |
| 난이도 | 중상 (Rust) |
| 차별성 | **세계 유일** |

저장소가 공식 지원합니다:

```rust
// crates/foundation/src/rule_pack.rs:36
pub trait RulePack: Send + Sync + std::fmt::Debug {
    fn check_text(&self, content, file_path, ext) -> Vec<Finding> { vec![] }
    fn check_element_dom(&self, dom, el) -> Vec<Finding> { vec![] }
    fn check_page_dom(&self, dom) -> Vec<Finding> { vec![] }
}
```

`docs/ENGINE.md`: "룰 팩은 **포크하지 않고** 자체 룰을 추가하는 방법이다.
팩이 없으면 아무것도 바뀌지 않으며, oracle이 바이트 단위로 강제한다."

만들 룰 후보:

| 룰 ID 안 | 잡는 문제 |
|---|---|
| `kr-tight-leading` | 한글 본문 `line-height < 1.6` |
| `kr-negative-tracking` | 한글 텍스트에 `letter-spacing < 0` |
| `kr-no-keep-all` | 한글 블록에 `word-break: keep-all` 누락 |
| `kr-tiny-text` | 한글 본문 `< 15px` |
| `kr-missing-webfont` | 한글인데 한글 웹폰트 미지정 (OS별 파편화) |
| `kr-overused-font` | Noto Sans KR / 본고딕 단독 사용 |
| `kr-line-length` | 한글 기준 줄 길이 |
| `kr-mixed-script-baseline` | 한글+라틴 혼용 시 베이스라인/크기 불일치 |
| `kr-all-caps-mix` | 한글에 `text-transform: uppercase` (무의미) |
| `kr-number-font` | 숫자 폴백으로 폭이 들쭉날쭉 |
| `kr-vertical-rhythm` | 수직 리듬이 라틴 기준 |
| `kr-josa-break` | 조사가 줄 끝에 혼자 떨어짐 |

확장: 같은 논리로 JP 팩, CN 팩. 아시아 시장 전체가 같은 공백을 갖고 있습니다.

#### 3위. AI 디자인 품질 감사 서비스 (B2B)

| 티어 | 내용 | 가격대 |
|---|---|---|
| 베이직 감사 | `detect` 전체 스캔 + 우선순위 리포트 | 50~100만원 |
| 풀 감사 | 위 + `critique`/`audit` AI 리뷰 + WCAG 수동 검증 + 경쟁사 비교 | 150~300만원 |
| 구축 포함 | 위 + `DESIGN.md` 수립 + CI 게이트 설치 + 팀 교육 | 500만원~ |
| 리테이너 | PR 자동 검사 + 월간 추이 + 신규 화면 리뷰 | 월 200~500만원 |

왜 팔리는가: "AI로 빨리 만들었는데 품질이 걱정"인 기업이 급증.
**객관적 근거**가 있습니다("61개 룰 중 23개 위반, 총 187건").
그리고 한국은 장애인차별금지법상 웹 접근성 의무가 있어 공공/금융이 특히 민감합니다.

타깃 순서: AI로 빠르게 만든 스타트업 → 에이전시(화이트라벨 재판매) →
공공/금융(규제, 단가 최고)

#### 4위. CI/CD 품질 게이트 SaaS (GitHub App)

```
PR 생성 → 변경된 UI 파일 감지 → impeccable detect --json (컨테이너)
       → PR 코멘트 "새 발견사항 3건"
       → 대시보드: 추이, 룰별 통계, 팀 순위, DESIGN.md 준수율
```

React로 만들 부분이 바로 여기입니다.
가격 경쟁력이 핵심: 디텍터가 LLM을 안 쓰므로 **변동비가 거의 0**.
LLM-as-judge 경쟁 제품은 PR마다 API 비용이 나갑니다.

차별화: 한글 룰 팩 내장 + `DESIGN.md` 준수율 추적.
"우리 토큰 밖의 색이 지난달 12개에서 3개로"는 경영진에게 보여줄 수 있는 지표입니다.

#### 5위. 61룰 통과 보장 스타터킷/템플릿

Next.js/React 랜딩, 대시보드, SaaS 템플릿을 61룰 전량 통과 +
한글 타이포 최적화 + `DESIGN.md` 완비 상태로 판매 ($29~99, 번들 $149~299).

차별 포인트: "61개 룰 전량 통과를 CI로 보증합니다."
검증 가능한 품질 주장이라 템플릿 시장에서 매우 희귀합니다.

#### 6위. 한국어 포크 배포 (유입 채널)

`skill/reference/*.md` 36개 한국어 번역 + 한글 룰 팩 번들 + 한국 사례로 교체.
직접 수익은 없지만 1~5번의 깔때기 입구가 됩니다.

주의: 이름 혼동 금지("Korean Edition" 같은 비공식 명시),
Apache-2.0 고지와 `NOTICE.md` 유지, 변경 사항 표시.
upstream은 issue-first 정책이므로 기여 시 이슈를 먼저 열어야 합니다.

#### 7위. 유료 뉴스레터 / 커뮤니티

매주 실제 사이트 1개 분석 + 개선안 / 새 AI 모델의 디자인 출력 비교 /
룰 해설 + 디자인 원리 / 릴리스 노트 한국어 해설 (월 $5~15).
디텍터가 무료이므로 소재가 무한합니다.

#### 8위. 산업별 수직 룰 팩

| 팩 | 룰 예시 | 타깃 |
|---|---|---|
| 금융/핀테크 | 금액 표기 일관성, `tabular-nums`, 필수 고지 가시성 | 은행, 증권 |
| 의료/헬스케어 | 고령자 가독성, 색맹 안전, 긴급 정보 위계 | 병원, 헬스앱 |
| **공공/접근성** | **한국 웹 접근성 지침(KWCAG) 매핑** | 공공기관 (규제 수요) |
| 이커머스 | 가격/할인 표기, CTA 위계, 장바구니 흐름 | 쇼핑몰 |
| 게임/엔터 | 고채도 UI 가독성, HUD 가림 | 게임사 |

공공/접근성 팩이 가장 유망합니다. 규제 기반 수요는 가격 저항이 낮고
KWCAG는 명문화돼 있어 룰로 옮기기 쉽습니다.

#### 9위. 내 AI 에이전트 제품에 품질 레이어로 내장

```
사용자 요청 → LLM 생성 → detect (WASM, 브라우저 내, 비용 0, 수 ms)
          → 발견사항 있으면 피드백 후 재생성 (최대 2회) → 전달
```

마케팅 문구: "우리는 생성된 모든 UI를 61개 디자인 룰로 검증합니다."
`crates/wasm --features detect`가 `detect_text_json` / `detect_html_source_json`을
JSON 익스포트로 노출하므로 바이너리 실행 없이 브라우저/엣지에서 직접 돌릴 수 있습니다.

#### 10위. 프리랜스 "AI 생성 UI 리디자인" 패키지

타깃: "Cursor/Claude로 MVP 만들었는데 보여주기 창피하다"는 1인 창업자, 소규모 팀.
건당 100~500만원. **현금화가 가장 빠릅니다.**

프로세스: `detect` 무료 진단 리포트 발송(영업 도구) → `critique` 리뷰 →
`polish`/`bolder`/`typeset` 수정 → `audit` 통과 확인 → `DESIGN.md` 인계

강점: 작업이 빠르고 **근거를 제시할 수 있습니다.**
전/후 디텍터 리포트가 성과 증빙이 됩니다. 일반 디자인 외주가 못 하는 것입니다.
무료 진단이 핵심 영업 장치입니다 (비용 0, 5분, 구체적 가치).

### 추천 로드맵

```
Phase 1 (1~3개월) 신뢰와 현금흐름
  #1 유튜브 시작 (주 1편, 시즌 1 먼저)
  #10 프리랜스 (무료 진단 영업으로 즉시 현금화)
  #6 한국어 번역 포크 시작
  목표: 구독자 1,000 / 외주 1~2건

Phase 2 (3~6개월) 제품화
  #2 한글 룰 팩 무료 5룰 공개
  #1 인프런 유료 강의 출시
  #5 스타터킷 1~2개
  #7 뉴스레터
  목표: 강의 수익 + 룰 팩 사용자 기반

Phase 3 (6~12개월) 확장
  #2 Pro 룰 팩 유료화 (+ JP/CN 검토)
  #3 B2B 감사 서비스
  #8 공공/접근성 팩 (규제 수요)
  #4 CI SaaS (확장성 최고, 가장 늦게)
```

### 리스크와 대응

| 리스크 | 심각도 | 대응 |
|---|---|---|
| upstream이 한글 지원 직접 추가 | 중 | 룰 팩 방식이면 오히려 호재. 교육/서비스는 영향 없음 |
| 상표/브랜드 혼동 | 중 | "비공식" 명시, Apache 고지 유지, "공식" 주장 금지 |
| AI 코딩 도구 시장 재편 | 중 | Impeccable 자체가 17개 하네스 추상화. "툴 중립 린터" 포지션 유지 |
| 한국 시장이 작음 | 높음 | 영어 콘텐츠 병행. 룰 팩과 SaaS는 글로벌 상품. JP/CN 시장이 훨씬 큼 |
| "그냥 린터 아니냐" 저평가 | 중 | 숫자로 말할 것 ("187건, 접근성 위반 23건") |
| upstream 업데이트 추적 부담 | 낮음 | 룰 팩 API로 포크 유지보수 최소화 |

### 가장 솔직한 추천

**지금 당장 2개:**

1. **#10 프리랜스 리디자인.** `npx impeccable detect`로 무료 진단 리포트를 만들어
   잠재 고객 10명에게 발송. 비용 0, 오늘 시작 가능, 가장 빠른 현금화.
2. **#1 유튜브 2번 영상** ("유명 사이트 디자인 점수 1분 만에 확인").
   진입 장벽 0, 바이럴 요소, 나머지 전부의 깔때기.

**가장 큰 기회: #2 한글 타이포그래피 룰 팩.**
61룰에 CJK가 완전히 비어 있고, 룰 팩 API가 공식 지원되고, 경쟁자가 없습니다.
이 세 조건이 동시에 성립하는 기회는 드뭅니다.

---

## 12. 참고 링크

| 항목 | URL |
|---|---|
| 내 저장소 (포크) | https://github.com/bmshin94/impeccable |
| 원본 저장소 | https://github.com/pbakaus/impeccable |
| 공식 사이트 | https://impeccable.style |
| 디텍터 문서 | https://impeccable.style/docs/detector |
| 훅 문서 | https://impeccable.style/docs/hooks |
| 사례 연구 (Neo Mirai) | https://impeccable.style/cases/neo-mirai |
| npm | https://www.npmjs.com/package/impeccable |
| VS Code 확장 | https://marketplace.visualstudio.com/items?itemName=renaissance-geek.impeccable |
| 참고: Anthropic frontend-design | https://github.com/anthropics/skills/tree/main/skills/frontend-design |
| 참고: 네이티브 레퍼런스 원본 (MIT) | https://github.com/ehmo/platform-design-skills |

### 저장소 내부 읽을 문서

| 파일 | 내용 |
|---|---|
| `CLAUDE.md` | 52KB. 프로젝트 전체 규칙과 설계 근거. **가장 먼저 읽을 문서** |
| `AGENTS.md` | 에이전트용 요약 가이드 |
| `docs/ENGINE.md` | 크레이트 지도, 룰 팩 계약 |
| `docs/CLI-CONTRACT.md` | 318KB. 모든 verb의 관측 가능 동작 명세 |
| `docs/STYLE.md` | 편집 지침 (AI 문체 금지 규칙의 근거) |
| `docs/DEVELOP.md` | 기여자 가이드, 빌드 방법 |
| `docs/HARNESSES.md` | 17개 하네스별 상세 |
| `docs/LIVE-REWRITE-PLAN.md` | 라이브 모드 설계 |
| `skill/SKILL.src.md` | 스킬 본문. 라우터 표와 공통 디자인 법칙 |
| `tests/skill-behavior/README.md` | 시나리오 목록과 베이스라인 |
| `tests/framework-fixtures/README.md` | 픽스처 스키마 |

---

*이 문서는 Claude Opus 5 (Claude Code)와의 대화를 통해 작성되었습니다.*
*AI 지원으로 작성된 문서임을 명시합니다.*
