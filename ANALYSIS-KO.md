# deepclaude 분석 및 활용 전략 (한국어)

> 이 문서는 `deepclaude` 저장소의 코드를 직접 읽고 분석한 결과와,
> 이를 바탕으로 한 학습·수익화 전략을 정리한 것입니다.
>
> - **이 저장소:** https://github.com/bmshin94/deepclaude
> - **원본 저장소:** https://github.com/aattaran/deepclaude (Ali Sharer)
> - 작성일: 2026-09-13

---

## 목차

1. [한 줄 요약](#1-한-줄-요약)
2. [저장소 구조](#2-저장소-구조)
3. [동작 원리](#3-동작-원리)
4. [설치 및 사용법](#4-설치-및-사용법)
5. [플러그인? 스킬? MCP?](#5-플러그인-스킬-mcp)
6. [API 토큰 정책](#6-api-토큰-정책)
7. [왜 이 장르가 뜨는가](#7-왜-이-장르가-뜨는가)
8. [로컬 에이전트 구축에 주는 가치](#8-로컬-에이전트-구축에-주는-가치)
9. [수익화 전략](#9-수익화-전략)
10. [React / PHP 로 다시 만든다면](#10-react--php-로-다시-만든다면)
11. [주의사항 및 리스크](#11-주의사항-및-리스크)
12. [참고 링크](#12-참고-링크)

---

## 1. 한 줄 요약

> **Claude Code의 자율 에이전트 루프(몸통)는 그대로 두고, 실제로 추론하는 모델(뇌)만 저렴한 백엔드로 교체하는 래퍼 도구.**

파일 읽기/쓰기, Bash 실행, Git 조작, 서브에이전트 생성 등 에이전트 기능은 전부 유지되고,
API 요청이 향하는 목적지만 DeepSeek / OpenRouter / Fireworks 로 바뀝니다.

---

## 2. 저장소 구조

총 7개 파일로 구성된 미니멀한 프로젝트입니다.

| 경로 | 역할 |
|---|---|
| `deepclaude.sh` | macOS / Linux 실행 스크립트 (핵심 런처) |
| `deepclaude.ps1` | Windows PowerShell 버전 (동일 기능) |
| `proxy/model-proxy.js` | **핵심 로직.** Node.js HTTP 프록시 서버 (443줄) |
| `proxy/start-proxy.js` | 프록시 실행 엔트리포인트 |
| `README.md` | 영문 설명서 |
| `proxy/README.md` | 프록시 기술 문서 |
| `screenshots/` | 데모 스크린샷 3장 |
| `CLAUDE.md` | 프로젝트 페르소나 설정 |

---

## 3. 동작 원리

### 3-1. 핵심 아이디어 — 환경변수 스위칭

Claude Code는 아래 환경변수들을 읽어 **어디로 API 요청을 보낼지** 결정합니다.

| 변수 | 역할 |
|---|---|
| `ANTHROPIC_BASE_URL` | API 엔드포인트 주소 |
| `ANTHROPIC_AUTH_TOKEN` | 인증 키 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Opus 티어 작업에 쓸 모델명 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Sonnet 티어 작업에 쓸 모델명 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Haiku 티어(서브에이전트) 모델명 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 서브에이전트 전용 모델 |

`deepclaude.sh:215-218` 에서 이 값들을 교체한 뒤 `exec claude` 로 실행합니다.

```bash
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="$DEEPSEEK_API_KEY"
export ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro"
exec claude "$@"
```

> 중요: **세션 한정**으로만 적용되며, 종료하면 원래 설정으로 돌아옵니다.
> 시스템 설정을 영구적으로 변경하지 않습니다.

### 3-2. 프록시(`model-proxy.js`)가 필요한 이유

단순 환경변수 교체만으로는 해결되지 않는 세 가지 문제가 있습니다.

#### (1) 리모트 컨트롤 트래픽 분할

`claude remote-control` 은 브라우저 연결용 WebSocket 주소가
`wss://bridge.claudeusercontent.com` 으로 **하드코딩**되어 있고 Anthropic OAuth 가 필요합니다.
`ANTHROPIC_AUTH_TOKEN` 을 DeepSeek 키로 바꾸면 이 브릿지 연결이 깨집니다.

```
claude remote-control
  ├── 브릿지 WebSocket → wss://bridge.claudeusercontent.com  (Anthropic, 고정)
  └── 모델 API 호출    → http://localhost:3200 (프록시)
                           ├── /v1/messages → DeepSeek
                           └── 그 외 전부   → Anthropic (패스스루)
```

#### (2) 세션 중 백엔드 실시간 전환

프록시에 컨트롤 엔드포인트를 노출해 재시작 없이 백엔드를 바꿉니다.

| 엔드포인트 | 기능 |
|---|---|
| `POST /_proxy/mode` | 백엔드 전환 (`backend=deepseek` 등) |
| `GET /_proxy/status` | 현재 백엔드 / 업타임 / 요청 수 |
| `GET /_proxy/cost` | 토큰 사용량 및 절감액 |

`~/.claude/commands/` 에 슬래시 커맨드를 등록하면 Claude Code 안에서
`/deepseek`, `/anthropic` 만 입력해 즉시 전환할 수 있습니다.

#### (3) 호환성 보정 — 실전 삽질의 기록

`model-proxy.js` 에서 가장 값진 부분입니다.

| 문제 | 해결 | 위치 |
|---|---|---|
| DeepSeek/OpenRouter 가 `usage` 필드를 누락해 Claude Code 크래시 | `UsageNormalizer` Transform 스트림으로 SSE 이벤트 가로채 기본값 주입 | `:40-91` |
| 모델명 불일치 (`claude-opus-4-6` 를 DeepSeek 이 모름) | `MODEL_REMAP` 테이블로 요청 바디의 모델명 자동 치환 | `:10-25` |
| 백엔드 전환 시 thinking 블록 때문에 400 에러 | 비-Anthropic 백엔드로는 전부 제거, Anthropic 복귀 시에도 외부에서 생성된 서명 블록 제거 | `:107-123`, `:337-354` |
| 경로 중복 (`/api/v1` + `/v1/messages` = `/v1/v1/messages`) | 접두사 겹침 자동 탐지 후 절단 | `:281-291` |
| 포트 충돌 | 3200부터 최대 20회 증가 재시도 | `:426-439` |
| 비용 추적 | 토큰 집계 후 Anthropic 대비 절감액 계산 | `:155-187` |

---

## 4. 설치 및 사용법

### 4-1. 준비물

| 필요한 것 | 확인 방법 |
|---|---|
| Claude Code CLI | `claude --version` |
| Node.js 18+ | `node --version` |
| DeepSeek API 키 | platform.deepseek.com 에서 발급 |

### 4-2. 설치 (macOS / Linux)

```bash
# 1) API 키 등록
echo 'export DEEPSEEK_API_KEY="sk-..."' >> ~/.bashrc
source ~/.bashrc

# 2) 실행 권한
chmod +x deepclaude.sh

# 3) PATH 등록
sudo ln -s "$(pwd)/deepclaude.sh" /usr/local/bin/deepclaude
```

### 4-3. 설치 (Windows)

```powershell
setx DEEPSEEK_API_KEY "sk-..."
Copy-Item deepclaude.ps1 "$env:USERPROFILE\.local\bin\deepclaude.ps1"
```

### 4-4. 첫 실행 순서 (권장)

```bash
deepclaude --status      # 1) 키 등록 확인
deepclaude --benchmark   # 2) 실제 연결 테스트  ← 가장 중요
deepclaude --cost        # 3) 가격표 확인
deepclaude               # 4) 실행
```

> `--benchmark` 를 먼저 돌려야 하는 이유: 스크립트가 `deepseek-v4-pro` 라는
> 모델명을 가정하고 있어, 제공자 쪽에서 이름이 바뀌었다면 바로 실패합니다.

### 4-5. 주요 명령어

```bash
deepclaude                     # DeepSeek (기본)
deepclaude -b or               # OpenRouter (미국 서버)
deepclaude -b fw               # Fireworks AI
deepclaude -b anthropic        # 원래 Claude Code
deepclaude --remote            # 브라우저/모바일 세션 URL 발급
deepclaude -s anthropic        # 실행 중 백엔드 전환
```

### 4-6. 지원 백엔드 (README 기준)

| 백엔드 | 플래그 | Input/M | Output/M | 서버 위치 |
|---|---|---|---|---|
| DeepSeek (기본) | `-b ds` | $0.44 | $0.87 | 중국 |
| OpenRouter | `-b or` | $0.44 | $0.87 | 미국 |
| Fireworks AI | `-b fw` | $1.74 | $3.48 | 미국 |
| Anthropic | `-b anthropic` | $3.00 | $15.00 | 미국 |

> 위 수치는 저장소 README 에 기재된 값이며 별도 검증되지 않았습니다.
> 실제 요금은 각 제공자의 공식 가격표를 확인하세요.

---

## 5. 플러그인? 스킬? MCP?

**셋 다 아닙니다.**

| 개념 | 정의 | 비유 |
|---|---|---|
| **MCP** | Claude 가 외부 도구에 접근하게 해주는 프로토콜 | 로봇에 새 손을 달아줌 |
| **스킬** | 특정 작업 수행 방법을 알려주는 지침 묶음 | 로봇에게 매뉴얼을 읽힘 |
| **플러그인** | 스킬 + 커맨드 + MCP 를 묶은 배포 패키지 | 부품 세트 박스 |
| **deepclaude** | 실행 환경을 교체하는 외부 셸 스크립트 + 로컬 프록시 | 로봇의 머리를 교체 |

즉 **Claude Code 내부에 설치되는 확장이 아니라, Claude Code 를 감싸 대신 실행하는 래퍼**입니다.

> 역설: deepclaude 를 사용하면 **MCP 가 오히려 동작하지 않습니다.**
> DeepSeek 호환 레이어가 MCP 툴 호출을 지원하지 않기 때문입니다.

---

## 6. API 토큰 정책

| 모드 | 필요한 인증 | 과금 |
|---|---|---|
| 기본 (`deepclaude`) | DeepSeek API 키 | 종량제 |
| OpenRouter (`-b or`) | OpenRouter API 키 | 종량제 |
| Fireworks (`-b fw`) | Fireworks API 키 | 종량제 |
| Anthropic (`-b anthropic`) | 기존 Claude Code 로그인 | 기존 구독/과금 |
| **원격 (`--remote`)** | **DeepSeek 키 + Anthropic OAuth 둘 다** | 종량제 + 구독 |

> `--remote` 는 브릿지가 Anthropic 인프라이므로 **claude.ai 구독이 여전히 필요합니다.**
> "구독을 해지하고 이것만 쓰면 된다"가 아닙니다.

---

## 7. 왜 이 장르가 뜨는가

deepclaude 개별 프로젝트가 아니라 **"Claude Code 를 저렴하게 쓰기"라는 카테고리 전체**가 성장 중입니다.

| 프로젝트 | 규모 |
|---|---|
| `musistudio/claude-code-router` | ⭐ 약 31,400 (카테고리 1위) |
| `aattaran/deepclaude` | 최근 급상승 (정확한 수치 미확인) |
| `claude-proxy`, `UniClaudeProxy` 등 | 다수 경쟁 |

### 성장 이유

1. **실제로 아픈 문제** — Claude Code 는 성능은 뛰어나지만 비싸고 사용량 제한이 있음
2. **공식적으로 열린 통로** — `ANTHROPIC_BASE_URL` 은 공식 지원 환경변수 (해킹이 아님)
3. **대체 모델의 급성장** — DeepSeek, Qwen, GLM, Kimi 등이 코딩 성능에서 추격
4. **수치로 표현되는 가치** — "17배 저렴"이라는 한 줄이 그대로 마케팅 카피
5. **deepclaude 만의 차별점** — 세션 중 전환, 비용 추적, 원격 세션, 스크린샷 데모

---

## 8. 로컬 에이전트 구축에 주는 가치

### 학습 교재로서 — 매우 높음

`model-proxy.js` 443줄에 LLM 프록시 구현 시 마주치는 실전 문제가 집약되어 있습니다.

| 배울 수 있는 것 | 실무 활용 |
|---|---|
| SSE 스트리밍 중계 및 가공 | 실시간 응답 UI 구현의 핵심 |
| 응답 스키마 정규화 | 다른 제공자 붙일 때 반드시 필요 |
| 모델명 매핑 테이블 패턴 | 멀티 프로바이더 라우팅 |
| 경로 접두사 충돌 처리 | 게이트웨이 구현 시 흔한 함정 |
| 토큰 집계 및 비용 산출 | 에이전트 운영의 필수 기능 |
| 포트 충돌 자동 회피 | 로컬 데몬의 실무 디테일 |

특히 thinking 블록 처리 주석은 **"왜 400 에러가 나는가"를 삽질로 알아낸 기록**이라 가치가 높습니다.

### 실전 도구로서 — 제한적

**적합한 경우**
- 에이전트 루프 반복 테스트 (호출 비용 부담이 낮음)
- 대량 리팩토링 / 코드 변환 배치 작업
- 동일 작업을 여러 모델로 비교하는 실험

**부적합한 경우**
- MCP 연동이 필요한 에이전트 (미지원)
- 비전/이미지 입력이 필요한 작업 (미지원)
- 팀 공유 또는 서버 배포 (1인용 설계)

> 결론: **읽어서 배우는 가치 > 그대로 쓰는 가치**

---

## 9. 수익화 전략

### 9-1. 시장 상황 (2026 기준)

| 지표 | 수치 |
|---|---|
| AI 비용 관리를 우선순위로 꼽은 기업 | **98%** (2025년 63% → 급증) |
| AI 비용 예측을 25% 이상 빗나간 기업 | **80%** |
| FinOps 팀이 추적 가능한 AI 지출 비율 | **40~60%** |
| 엔지니어 1인당 월 AI 코딩 지출 | $150~250 (헤비 유저 ~$2,000) |
| 참고 사례 | Uber 가 2026년 AI 코딩 예산을 4월에 소진 |
| 참고 사례 | Copilot 과금 개편으로 $29 → $750, $50 → $3,000 사례 발생 |
| 시장 변화 | **Helicone 이 2026-03 Mintlify 에 인수 후 유지보수 모드 전환** |

핵심 통찰:

> deepclaude 는 **"싸게 쓰는 법"** 을 해결했습니다.
> 그러나 기업이 실제로 돈을 지불하는 대상은 **"어디로 새는지 보이게 하고 통제하는 법"** 입니다.

### 9-2. 아이디어 5종

#### A. 콘텐츠 및 교육 — 난이도 ★ / 즉시 시작

| 형태 | 가격대 |
|---|---|
| 블로그 / 유튜브 | 무료 (리드 확보용) |
| 전자책 | 2~3만원 |
| 온라인 강의 | 8~15만원 |
| 기업 사내 세미나 | 회당 50~150만원 |

한국어 자료가 거의 없는 분야로, 선점 효과가 큽니다.
목적은 직접 수익보다 **신뢰와 리드 확보**입니다.

#### B. 팀용 AI 비용 대시보드 — 난이도 ★★★ / 본진 ⭐

해결할 문제: *"이번 달 AI 비용이 왜 이렇게 나왔고, 누가 얼마나 썼는가?"*

**MVP 범위**

| 기능 | 기술 |
|---|---|
| 게이트웨이 (요청 중계) | Node.js — deepclaude 프록시 응용 |
| 사용자별 사용량 추적 | PHP + MySQL |
| 실시간 대시보드 | React |
| 예산 알림 (Slack / 카카오) | PHP 크론 |
| 한도 초과 시 자동 차단 | Node 미들웨어 |

**Phase 2 차별화**
- 작업 난이도 기반 **자동 모델 다운그레이드** (deepclaude 지식이 직접 활용되는 지점)
- 레포지토리 / 프로젝트별 비용 귀속
- 월간 절감액 리포트 (구독 갱신 설득 자료)

**가격 전략 (시장 벤치마크 반영)**

| 참고: 경쟁사 | 가격 |
|---|---|
| Helicone Pro / Team | $79 / $799 per month |
| Portkey, LiteLLM 매니지드 | 약 $49/월부터 |
| LiteLLM, Langfuse 셀프호스팅 | 무료 (오픈소스) |

| 제안 티어 | 가격 | 범위 |
|---|---|---|
| Free | $0 | 5명, 7일 보관 |
| Pro | $49/월 | 20명, 3개월 보관, 알림 |
| Business | $199/월 | 무제한, 1년 보관, 온프레미스 |

목표 감각: $49 × 100팀 = 월 약 650만원.

#### C. Helicone 이탈 고객 유치 — 난이도 ★★ / 기간 한정

Helicone 이 유지보수 모드로 전환되어 기존 사용자들이 대안을 찾는 시점입니다.

- "Helicone 대안" 키워드 콘텐츠 (한/영)
- 데이터 마이그레이션 도구 제공
- 무료 이전 지원 → B 구독으로 전환
- 마이그레이션 컨설팅: 건당 100~300만원

> 타이밍 사업입니다. 6개월 내 실행하지 않으면 기회가 사라집니다.

#### D. 기업 컨설팅 및 구축 대행 — 난이도 ★★ / 즉시 시작

| 단계 | 내용 | 가격 |
|---|---|---|
| 진단 | 현재 AI 비용 분석 + 절감 가능액 리포트 | 200~500만원 |
| 구축 | 게이트웨이 + 대시보드 + 정책 수립 | 500~2,000만원 |
| 운영 | 월간 리포트 + 최적화 + 지원 | 월 100~300만원 |

**성과 기반 과금**(절감액의 일정 %)이 고객 리스크를 없애 성사율이 높습니다.
선투자 없이 시작 가능하며, **고객의 요구사항이 곧 B 제품의 설계도**가 됩니다.

#### E. 규제 특화 온프레미스 게이트웨이 — 난이도 ★★★★★ / 장기

대상: 금융 / 의료 / 공공 / 방산 — 코드의 외부 전송이 금지된 업계.

| 판매 항목 | 근거 |
|---|---|
| 완전 온프레미스 설치 | 클라우드 SaaS 는 검토 대상에서 제외됨 |
| 민감정보 자동 마스킹 | 개인정보가 프롬프트에 섞여 나가는 위험 |
| 전체 감사 로그 | 감사 대응 필수 |
| 국내 서버 라우팅 강제 | 데이터 국외이전 규제 |

계약 규모 연 3,000만원~2억원. 단 영업 사이클 6개월~1년, SI 파트너십 사실상 필수.

### 9-3. 90일 실행 플랜

```
1개월차 — 씨앗 뿌리기
  ├─ deepclaude 분석 글 발행 (이 문서 활용)
  ├─ "AI 비용 관리" 주제 3~5편 추가 작성
  ├─ 개발자/팀장 10명 인터뷰  ← 가장 중요
  │    질문: 월 AI 비용은? 누가 썼는지 아는가? 무엇이 불편한가?
  └─ 랜딩페이지 + 대기자 명단 수집

2개월차 — 검증 및 첫 수익
  ├─ 인터뷰에서 검증된 고통 1개를 MVP 범위로 확정
  ├─ Node 프록시 + PHP 기록 + React 대시보드 최소 버전
  ├─ 무료 진단(아이디어 D) 2~3곳 진행
  └─ 첫 유료 계약 시도

3개월차 — 제품화
  ├─ 베타 팀 5곳 확보 및 피드백 수집
  ├─ "Helicone 대안" 콘텐츠로 해외 유입 시도
  └─ 유료 전환 시작
```

> **핵심 원칙: 코드보다 인터뷰가 먼저입니다.**
> 10명 중 7명이 강하게 공감하면 진행, 2명 수준이면 피벗하세요.

### 9-4. 경쟁 지형과 진입 틈새

| 경쟁자 | 강점 | 빈틈 |
|---|---|---|
| LiteLLM | 무료 오픈소스, 강력 | 개발자 전용, 비개발자가 못 씀 |
| Portkey | 기능 풍부, ~$49부터 | 한국어/국내 결제 미지원 |
| Langfuse | 오픈소스, 관측 특화 | 디버깅 중심, 비용 통제는 약함 |
| Helicone | 시장 점유 있었음 | **유지보수 모드 — 이탈 진행 중** |
| claude-code-router | ⭐31k, 라우팅 강력 | 1인용, 팀 기능 없음 |

**공략 가능한 틈 3가지**

1. **코딩 에이전트 특화** — 기존 도구는 범용 LLM 앱 대상. 에이전트 루프는 한 작업에 수십 회 호출되어 비용 패턴이 완전히 다름
2. **비개발자용 화면** — 팀장/재무가 보고 이해할 수 있는 대시보드 (React 강점 구간)
3. **한국 시장** — 한국어 UI, 세금계산서, 국내 서버, 카카오 알림, 국내 결제

---

## 10. React / PHP 로 다시 만든다면

| 결론 | 이유 |
|---|---|
| React 로 프록시 구현 | ❌ 불가 — 브라우저에서 직접 호출 시 CORS 차단 + API 키 노출 |
| PHP 로 프록시 구현 | ⚠️ 가능하나 비권장 — SSE 스트리밍/장기 연결에 취약 |
| **React + PHP + Node 조합** | ⭕ **권장** |

### 기술 비교

| 항목 | PHP | Node.js |
|---|---|---|
| 기본 요청/응답 중계 | 쉬움 | 쉬움 |
| 실시간 스트리밍(SSE) | 버퍼링 이슈로 난이도 높음 | 태생적으로 적합 |
| 장기 연결 (5분+) | 타임아웃 관리 필요 | 문제 없음 |
| 동시 요청 처리 | 프로세스 점유 | 이벤트 루프로 경량 처리 |

### 권장 아키텍처

```
┌─────────────────────────────────────────┐
│  React — 대시보드 (프론트엔드)             │
│  · 실시간 비용 그래프                      │
│  · 모델 전환 UI                           │
│  · 요청 로그 타임라인                      │
│  · 팀원별 사용량 순위                      │
└──────────────┬──────────────────────────┘
               │ REST API
┌──────────────▼──────────────────────────┐
│  PHP (Laravel) — 관리 서버                │
│  · 인증 / 권한                            │
│  · 사용 기록 DB 저장                       │
│  · 예산 한도 및 알림                       │
│  · 통계 집계                              │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  Node.js — 프록시 (deepclaude 응용)        │
│  · 실제 모델 요청 중계 + SSE 스트리밍       │
│  · 토큰 집계 후 PHP 로 기록 전송            │
└─────────────────────────────────────────┘
```

핵심 스트리밍 계층만 Node 로 두고 나머지를 React + PHP 로 구성하면,
기존 역량을 그대로 활용하면서 전체의 약 90% 를 직접 구현할 수 있습니다.

---

## 11. 주의사항 및 리스크

### 기술적 제약

| 제약 | 영향 |
|---|---|
| 이미지/비전 입력 미지원 | 스크린샷 기반 디버깅 불가 |
| MCP 서버 툴 미지원 | 외부 연동 에이전트 구성 불가 |
| Anthropic `cache_control` 무시 | 프롬프트 캐싱 전략이 다름 |
| `CLAUDE_CODE_EFFORT_LEVEL="max"` 하드코딩 (`deepclaude.sh:85`) | 항상 최대 추론으로 동작해 토큰 소비 증가 — 절약이 목적이면 조정 권장 |

### 데이터 및 규정

- **DeepSeek 기본 엔드포인트는 중국 서버입니다.** 사내 코드나 민감 정보는 전송 금지.
  필요 시 `-b or` (OpenRouter, 미국 서버) 사용을 검토하세요.
- `--remote` 는 Anthropic 구독 브릿지를 사용하면서 모델 호출만 외부로 보내는 구조입니다.
  **개인 사용과 별개로, 이를 상업적으로 판매하는 것은 약관 위반 소지가 큽니다.**

### 수익화 시 피해야 할 것

| 금지 사항 | 이유 |
|---|---|
| 이 코드를 그대로 재포장해 유료 판매 | MIT 라이선스상 가능하나 원저작자 존재, 무료 경쟁자 다수 |
| `--remote` 기능의 상업적 판매 | 약관 위반 리스크 |
| "중국 모델이 저렴하다"만 소구 | 기업은 데이터 국외이전 때문에 오히려 거부감 |
| 고객 API 키를 자사 서버에 저장 | 보안 사고 시 치명적 — 키는 고객 환경에, 메타데이터만 수집 |
| 초기부터 대기업 타겟팅 | 영업 사이클 1년 — 10~50인 규모부터 공략 |

### 검증되지 않은 정보

- README 의 가격표와 "17배 저렴", "LiveCodeBench 96.4%" 등의 수치는 저장소 주장이며 별도 검증하지 않았습니다.
- `deepseek-v4-pro` 등 모델명이 실제 API 에 존재해야 동작합니다. 반드시 `--benchmark` 로 선확인하세요.

---

## 12. 참고 링크

### 저장소

- 이 저장소: https://github.com/bmshin94/deepclaude
- 원본 (aattaran/deepclaude): https://github.com/aattaran/deepclaude
- 동일 사본 (derlg-com/deepclaude-cli): https://github.com/derlg-com/deepclaude-cli

### 경쟁 / 유사 프로젝트

- claude-code-router (⭐31k): https://github.com/musistudio/claude-code-router
- claude-proxy: https://github.com/sunflower0305/claude-proxy
- UniClaudeProxy: https://github.com/vibheksoni/UniClaudeProxy

### 시장 조사 출처

- 2026 AI 비용 거버넌스 리포트: https://www.mavvrik.ai/blog/blog-ai-cost-governance-report-2026/
- AI 비용 가시성 격차: https://www.pointfive.co/blog/why-companies-overspend-on-ai-the-2026-cost-visibility-gap
- AI 코딩 비용 추적 가이드: https://larridin.com/blog/track-ai-coding-costs-by-team
- AI 코딩 어시스턴트 가격/ROI: https://getdx.com/blog/ai-coding-assistant-pricing/
- LLM 관측 도구 비교: https://insights.nomadlab.cc/blog/2026/05/langfuse-helicone-portkey-litellm-openrouter-2026
- LLM 게이트웨이 비교: https://agentscamp.com/guides/advanced/llm-gateways-compared
- AI 코딩 지출 거버넌스 프레임워크: https://weilliptic.ai/blog/ai-coding-spend-governance-a-framework-for-engineering-and-finance-leaders/

---

## 요약 표

| 질문 | 답 |
|---|---|
| 이게 뭐야? | Claude Code 를 감싸 모델 백엔드만 교체하는 래퍼 + 로컬 프록시 |
| 플러그인/스킬/MCP? | 모두 아님. 외부 셸 스크립트 + Node 프록시 (오히려 MCP 사용 불가) |
| API 키 필요? | 필수. DeepSeek 종량제. `--remote` 는 Anthropic 로그인도 추가 필요 |
| 왜 유명? | 장르 자체가 급성장. 1위는 ⭐31k 의 claude-code-router |
| 에이전트 공부에 도움? | 학습 교재로는 최상급, 실전 도구로는 제약 많음 |
| 수익화? | 코드 자체는 ❌ → 콘텐츠 → 컨설팅 → 팀 대시보드 → 규제 특화 순 |
| React/PHP 가능? | 프록시는 Node 유지, 대시보드는 React + PHP 조합 권장 |
