# 이도현 · Fullstack Architect & AI Agent Engineer

> **"복잡한 시스템을 AI 에이전트가 이해하고, 검수하고, 진화시킬 수 있도록 설계한다."**

MSA · Micro Frontends · AI Agent Orchestration을 관통하는 아키텍처를 설계하고,
셀프 호스팅 K3s 클러스터 위에서 인그레스·인증·Observability·백업까지 직접 운영합니다.

---

## Architecture Philosophy

- **End-to-End Ownership** — 도메인 모델링 → API 설계 → 프론트엔드 → 인프라 → 모니터링까지 수직 관통
- **AI-Native Development** — 코드를 작성하는 것이 아니라, 코드를 작성·검수·배포하는 **에이전트 시스템**을 설계
- **Design Decision as Code** — 모든 아키텍처 의사결정을 `AGENTS.md`(캐논)와 GitHub Issues/Projects로 코드화하여 추적. 여러 AI 도구·세션이 동시에 작업하는 환경이라, 진행 상황은 로컬 파일이 아니라 이슈 코멘트(Claim/Progress/Handoff)에 남겨 도구 간 인계가 항상 가능하게 함

---

## 🏗️ PosSelect — Micro Frontends × MSA E-Commerce Platform

풀스택 마이크로서비스 및 마이크로 프론트엔드 아키텍처 기반의 커머스 플랫폼.
10개 독립 레포지토리(프론트 4 · 공유 UI/셸 2 · API 3 · 게이트웨이 1)가 빌드 타임이 아니라
**런타임에** 통합됩니다. 여기에 아키텍처 문서 사이트와 이 프로필 저장소를 더한 12개를 운영합니다.

### System Architecture

```
                        ┌─────────────────────────────────────────┐
                        │          posselect-shell (MFE Host)     │
                        │   Runtime Federation / Module Stitching │
                        └────┬──────┬──────┬──────┬──────────────┘
                             │      │      │      │
                    ┌────────┘  ┌───┘  ┌───┘  ┌───┘
                    ▼           ▼      ▼      ▼
              customer.    product. admin.  store.
               front       front   front   front
                    │           │      │      │
                    └─────┬─────┘──────┘──────┘
                          ▼
               ┌─────────────────────┐
               │  @posselect/ui      │
               │  Design System      │
               │  (Git Dependency)   │
               └─────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
   │  auth.api   │ │ product.api │ │  order.api  │
   │  (Keycloak  │ │  (Spring)   │ │  (Spring)   │
   │   + JWT)    │ │             │ │             │
   └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
          └───────────────┼───────────────┘
                          ▼
               ┌─────────────────────┐
               │  Spring Cloud       │
               │  Gateway            │
               │  (WebFlux / Netty)  │
               └──────────┬──────────┘
                          ▼
               ┌─────────────────────┐
               │  K3s Cluster        │
               │  Traefik Ingress    │
               │  + MetalLB LB       │
               └─────────────────────┘
```

### Core Engineering Decisions

| 영역 | 설계 결정 | 근거 |
|------|----------|------|
| **Frontend Architecture** | Micro Frontends — `shell` 기반 런타임 통합 | 도메인 팀 단위 독립 배포, 빌드 격리, 장애 전파 차단 |
| **Design System** | `@posselect/ui` — Git 의존성 + `transpilePackages` | npm 레지스트리 없이 소스 레벨 공유, 디자인 토큰 동기화 보장 |
| **UI 검증** | Storybook `play` 함수 기반 인터랙션 테스트 | 시각적 회귀 방지 + 컴포넌트 단위 자동화 검증 |
| **API Gateway** | Spring Cloud Gateway (WebFlux/Netty) | 논블로킹 I/O 기반 리버스 프록시, JWT 직접 검증 아키텍처 |
| **인증** | Keycloak + JWT 토큰 직접 검증 | Gateway 레벨 SSO, 백엔드 서비스 무상태(Stateless) 유지 |
| **데이터 계층** | PostgreSQL + MySQL + Redis (서비스별 분리) | Polyglot Persistence — 도메인 특성에 맞는 스토리지 선택 |
| **오브젝트 스토리지** | MinIO (S3 호환) | 이미지/에셋의 자체 호스팅, imgproxy 연동 실시간 리사이징 |

---

## 🤖 AI Agent Harness Engineering

이기종 AI 코딩 도구(Claude Code · Codex · Antigravity)가 동일한 코드베이스에서 동시에 작업하는 것을
전제로 **에이전트 하네스**를 설계·운영합니다. 설계 목표는 두 가지입니다 — 어떤 도구로 작업하든 동일한
품질 기준이 적용될 것, 그리고 그 기준이 문서 규약에 머무르지 않고 **실행 가능한 게이트**로 존재할 것.

규약(Canon) · 도메인 검수(Sub-agent) · 결정론적 게이트(Gate)의 3계층으로 분리하고, 상위 계층의 판단이
누락되더라도 하위 계층이 받아내도록 구성했습니다.

### Harness Architecture

```
┌─ Layer 3 · Cross-Tool Canon ─────────────────────────────────────┐
│  AGENTS.md — 도구 무관 공통 규약 (11개 서비스 레포지토리에 배치) │
│  Check & Claim · Worktree Isolation · Handoff Protocol           │
└────────────────────────────────┬─────────────────────────────────┘
                                 │  규약 (convention)
┌─ Layer 2 · Domain Sub-agents ──▼─────────────────────────────────┐
│  레포지토리별 실패 모드에 대응하는 도메인 특화 검수자            │
│  Frontend · Backend · Infra — 커밋 전 자동 위임                  │
└────────────────────────────────┬─────────────────────────────────┘
                                 │  권고 (advisory)
┌─ Layer 1 · Deterministic Gates ▼─────────────────────────────────┐
│  verify.sh — git hook · agent hook · CI 가 공유하는 단일 진입점  │
│  token-mirror · i18n/diagram · architecture-drift · 회귀 테스트  │
│  LLM 비의존 · 우회 불가 (enforced)                               │
└──────────────────────────────────────────────────────────────────┘
```

### Enforcement Model

| 계층 | 구현 | 실행 시점 | 강제력 |
|------|------|-----------|--------|
| **Deterministic Gates** | 셸 스크립트 · 회귀 테스트 (LLM 비의존) | pre-push · CI | 우회 불가 |
| **Domain Sub-agents** | 레포지토리별 특화 검수 에이전트 | 커밋 전 위임 | 권고 |
| **Cross-Tool Canon** | `AGENTS.md` 공통 규약 | 세션 시작 시 로드 | 규약 |

**Deterministic Gates**

1. **Pre-push Verification** — `verify.sh` 단일 진입점을 git hook · 에이전트 훅 · CI가 공유.
   `main` push가 곧 프로덕션 배포인 파이프라인이므로, 검증을 CI에만 두지 않고 push 시점에서 차단
2. **Design Token Mirror Check** — 이원화된 토큰 정의 간 불일치와, 미정의 참조로 런타임에 조용히
   무효화되는 CSS 변수를 편집 직후 검출
3. **i18n / Diagram Integrity Check** — 특정 언어에만 누락되어 에러 없이 빈 문자열로 렌더링되는 키와,
   파손된 다이어그램 정의를 검출
4. **Architecture Drift Detection** — 라이브 클러스터의 Ingress 목록과 아키텍처 문서를 주기적으로
   대조하여 문서-실제 간 괴리를 검출
5. **Public Path Regression Test** — 인증 화이트리스트를 회귀 테스트로 고정. 경로 누락이 프로덕션이
   아니라 빌드 단계에서 실패하도록

**Domain Sub-agents**

| Sub-agent | 검수 대상 |
|-----------|-----------|
| `ui-token-guard` | 디자인 토큰 정합성 |
| `shell-contract-guard` | 런타임 셸 v1 계약 호환성 |
| `flyway-migration-guard` | 엔티티 ↔ 마이그레이션 정합성 |
| `tx-idempotency-reviewer` | 트랜잭션 전파 · 쓰기 멱등성 |
| `cache-invalidation-guard` | 캐시 무효화 경로 |
| `gateway-route-guard` | 인증 화이트리스트 정합성 |

**Cross-Tool Canon**

- **Check & Claim + Worktree Isolation** — 동시 세션 환경에서 작업의 중복 착수와 공용 클론의
  물리적 충돌을 방지
- **Handoff Protocol** — 진행 상태를 GitHub Issue에 구조화된 형식으로 기록. 도구별 로컬 메모리는
  타 도구가 참조할 수 없으므로, 이슈를 도구 간 인계의 단일 매체(SSOT)로 고정
- **Cross-Repository Impact Analysis** — 공유 컴포넌트 · API 스키마 변경 시 참조 레포지토리 사전 탐색.
  런타임 셸 변경은 4개 프론트엔드에 동시 반영되므로 특히 강제

---

## 🔭 Observability & Infrastructure

### Self-hosted K3s Cluster (`leedohyun.com`)

프로덕션 워크로드를 운영하는 셀프 호스팅 Kubernetes 클러스터.
모니터링·보안·DNS까지 외부 매니지드 서비스 없이 직접 설계·운영합니다.

```
┌─ Observability ──────────────────────────────────┐
│  Grafana   ◄── Prometheus / Alertmanager         │
│  Jaeger    ◄── OpenTelemetry 자동 계측            │
│  Loki      ◄── Promtail (Traefik accesslog)      │
└──────────────────────────────────────────────────┘

┌─ Data Layer ─────────────────────────────────────┐
│  PostgreSQL (×3)  │  MySQL  │  Redis (×2)        │
│  MinIO (S3)       │  imgproxy (실시간 리사이징)    │
└──────────────────────────────────────────────────┘

┌─ Security & Networking ──────────────────────────┐
│  Traefik Ingress  │  MetalLB LoadBalancer        │
│  Keycloak SSO     │  cert-manager (Let's Encrypt)│
│  Spring Cloud     │  Velero Backup               │
│   Gateway (단일    │  Route 53 DNS                │
│   진입점·인증)     │  SPF -all / DMARC reject     │
│  DDNS (동적 IP)    │  WAN 포트 최소화 (HTTP/S/SSH) │
└──────────────────────────────────────────────────┘
```

> **서비스 메시는 의도적으로 도입하지 않았습니다.** 이 규모에서 Istio는 운영 비용 대비 이득이
> 없다고 판단해, east-west 제어는 쇼핑몰 네임스페이스의 `default-deny-ingress` NetworkPolicy와
> 서비스별 allow 규칙으로, north-south 제어는 게이트웨이 단일 진입점으로 처리합니다.
> 분산 트레이싱은 메시 없이 OpenTelemetry 자동 계측으로 확보했습니다.

### Network Security Architecture

- **Attack Surface Minimization** — WAN 노출 포트를 HTTP/HTTPS/SSH로 한정, 불필요한 공격 벡터 제거
- **DNS Spoofing Prevention** — SPF `-all` + DMARC `reject` 정책으로 이메일 스푸핑 원천 차단
- **Dynamic IP Resilience** — DDNS 기반 도메인 운영으로 가정용 네트워크 환경에서도 안정적 서비스 제공
- **Supply Chain Security** — Dependabot 의존성 자동 업데이트 + Trivy 컨테이너 이미지 취약점 스캔을 11개 서비스 레포지토리 전체에 적용

---

## 🛠️ Tech Stack

**Frontend** &nbsp;
![Next.js](https://img.shields.io/badge/Next.js-000?logo=nextdotjs&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?logo=react&logoColor=000)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![Storybook](https://img.shields.io/badge/Storybook-FF4785?logo=storybook&logoColor=white)

**Backend** &nbsp;
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?logo=springboot&logoColor=white)
![Spring Cloud Gateway](https://img.shields.io/badge/Spring_Cloud_Gateway-6DB33F?logo=spring&logoColor=white)
![Keycloak](https://img.shields.io/badge/Keycloak-4D4D4D?logo=keycloak&logoColor=white)

**Infrastructure** &nbsp;
![K3s](https://img.shields.io/badge/K3s-FFC61C?logo=k3s&logoColor=000)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Traefik](https://img.shields.io/badge/Traefik-24A1C1?logo=traefikproxy&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?logo=helm&logoColor=white)

**Observability** &nbsp;
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)
![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-000?logo=opentelemetry&logoColor=white)
![Jaeger](https://img.shields.io/badge/Jaeger-66CFE3?logo=jaeger&logoColor=000)
![Loki](https://img.shields.io/badge/Loki-F5A800?logo=grafana&logoColor=000)

**Data** &nbsp;
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?logo=mysql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-FF4438?logo=redis&logoColor=white)
![MinIO](https://img.shields.io/badge/MinIO-C72E49?logo=minio&logoColor=white)

**AI Engineering** &nbsp;
![Claude](https://img.shields.io/badge/Claude_Code-191919?logo=anthropic&logoColor=white)
![Antigravity](https://img.shields.io/badge/Antigravity_IDE-4285F4?logo=google&logoColor=white)

---

## Engineering Principles

```
근본 원인을 찾아 구조적으로 정리한다.
되돌리기 어려운 작업일수록 정리 → 전환 → 기록의 순서를 지킨다.
AI는 도구가 아니라 아키텍처의 일부다.
```
