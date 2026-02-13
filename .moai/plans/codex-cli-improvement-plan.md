# MoAI-ADK Codex CLI 개선 실행계획 (리뷰 반영본)

## 0. 목적과 원칙

이 문서는 "Claude Code 중심 구조를 깨지 않고" Codex CLI를 1급 실행 타겟으로 추가하기 위한 **실행 가능한 계획**이다.

- 하위 호환 우선: 기존 Claude 사용자 기본 동작은 유지
- 점진적 확장: Provider 추상화 도입 후 기능 확장
- 테스트 선행: provider matrix로 회귀를 조기 차단

---

## 1. 현재 코드베이스 기준 진단

### 1.1 이미 준비된 기반

- `moai init`는 wizard + 템플릿 배포 구조를 갖고 있어 provider 선택지를 추가하기 좋은 위치다.
- hook 시스템은 이벤트 타입/입출력 구조가 정리되어 있어 provider adapter 계층을 얹기 쉽다.
- statusline은 Builder/Renderer 분리 구조라 provider별 입력 해석기를 독립 구현하기 적합하다.

### 1.2 실제 병목

1. **문구/프로토콜 결합**
   - `init` 명령 설명과 statusline 데이터 필드가 Claude 용어/입력에 결합되어 있음.
2. **출력 경로 결합**
   - `.claude/*` 출력 중심 설계(`cc`, `glm` 포함)로 provider 분기점이 명시적이지 않음.
3. **검증 경로 부재**
   - CI에서 provider 축(claude/codex) 분리 검증이 아직 계획 수준임.

---

## 2. 목표 상태 (Definition of Done)

### 제품 DoD

- `moai init`에서 `--provider` 플래그와 wizard 선택으로 `claude|codex`를 명시 가능
- 동일 명령 UX(`plan/run/sync`)를 유지하면서 provider별 실행 adapter 동작
- README EN/KO에 provider 선택 Quick Start 반영

### 기술 DoD

- provider registry + capability 모델 도입
- hook payload 변환 계층(공통 모델 ↔ provider 포맷) 도입
- statusline 입력 해석 계층(Claude/Codex) 분리
- GitHub Actions provider matrix 테스트 추가

### 품질 DoD

- Claude 경로 회귀 0
- Codex 최소 시나리오(E2E smoke) 통과
- 신규 코드(Provider 관련) 라인 커버리지 80%+

---

## 3. 변경 범위 (파일 단위)

## 3.1 1순위 (코어 경로)

- `internal/cli/init.go`
  - `--provider` 플래그 추가
  - non-interactive 모드에서 provider 검증 로직 추가
- `internal/cli/wizard/questions.go`
  - provider 질문 추가 + 기본값(claude) 명시
- `internal/template/deployer.go`
  - provider별 템플릿 루트 선택 로직 추가
- `internal/hook/types.go`, `internal/hook/contract.go`, `internal/hook/protocol.go`
  - 공통 HookEvent 모델과 provider 변환기 경계 정의
- `internal/statusline/builder.go`
  - 입력 JSON 해석을 provider adapter로 위임

## 3.2 2순위 (정책/운영)

- `internal/template/model_policy.go`
  - provider capability 기반 모델 정책 분기
- `internal/cli/cc.go`, `internal/cli/glm.go`
  - Claude 전용 동작임을 명시하고 provider 가드 추가
- `.github/workflows/ci.yml`
  - `PROVIDER=claude|codex` matrix 도입

## 3.3 문서

- `README.md`, `README.ko.md`
  - "Claude 전용" 인상을 줄이고 provider 선택 Quick Start 추가
- `CLAUDE.md`
  - 문서 범위를 "Claude 전용 운영 가이드"로 명확화

---

## 4. 단계별 실행 계획 (6주)

## Phase 1 (Week 1): Provider 골격 도입

**작업**
- `internal/provider` 패키지 신설
- `Provider` 인터페이스 + registry 추가
- 설정(`.moai/config/sections/system.yaml`)에 provider 키 추가

**수용 기준**
- 기본 provider=claude로 기존 동작 동일
- 빌드/기존 테스트 통과

## Phase 2 (Week 2): init/wizard/template 분기

**작업**
- `moai init --provider codex` 지원
- wizard에서 provider 질문 추가
- provider별 템플릿 루트(`providers/claude`, `providers/codex`) 도입

**수용 기준**
- init 골든 테스트에서 provider별 산출물 스냅샷 분리

## Phase 3 (Week 3-4): hook/statusline adapter

**작업**
- hook 공통 이벤트 모델 정의
- Claude 기존 포맷 adapter 유지 + Codex 포맷 adapter 추가
- statusline 입력 파서 분리

**수용 기준**
- hook protocol 테스트를 provider matrix로 실행
- statusline 렌더 결과 회귀 없음

## Phase 4 (Week 5): CI/품질 게이트

**작업**
- CI matrix 추가 (`PROVIDER=claude|codex`)
- 최소 E2E smoke: `init -> (mock) plan/run/sync` 경로 점검

**수용 기준**
- claude/codex 모두 필수 잡 녹색

## Phase 5 (Week 6): 문서/릴리즈

**작업**
- README EN/KO 업데이트
- 마이그레이션 가이드(기본 claude, codex opt-in) 배포
- 릴리즈 단계(Experimental -> Beta) 정의

**수용 기준**
- 신규 사용자 가이드만 보고 provider 선택 초기화 가능

---

## 5. 리스크와 차단 전략

1. **프로토콜 차이로 인한 추상화 과적합**
   - 대응: capability flag(`supports_hooks`, `supports_statusline`, `supports_agent_teams`)로 분리
2. **기존 Claude 사용자 설정 파손**
   - 대응: `moai update`에 dry-run + 백업 강제
3. **문서 다국어 유지 비용 증가**
   - 대응: EN 원문 우선 + KO는 릴리즈 마일스톤에 동기화

---

## 6. 이번 스프린트 즉시 실행 항목 (우선순위)

1. `init/wizard`에 provider 입력 경로 추가
2. `template/deployer`에 provider 루트 분기 추가
3. `hook` 공통 모델 인터페이스만 먼저 도입(구현은 Claude adapter pass-through)
4. CI에 provider matrix 골격 추가(초기 codex job은 smoke-only)
5. README EN/KO 상단 Quick Start를 provider 선택형으로 수정

---

## 7. 검증 커맨드 초안

아래 커맨드를 완료 기준으로 사용한다.

```bash
# 1) 정적/유닛 테스트
go test ./...

# 2) init provider 시나리오 (예시)
moai init sample-claude --provider claude --non-interactive
moai init sample-codex --provider codex --non-interactive

# 3) 회귀 확인
moai --help
moai hook --help
```

> 참고: 실제 codex 경로 검증은 provider adapter 구현 완료 후 테스트 픽스처로 고정한다.
