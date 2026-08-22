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

단순히 AI를 "사용"하는 것이 아니라, **12개 레포지토리에 걸쳐 실제로 실행되는 에이전트 하네스**를
설계했습니다. 핵심 교훈은 **선언이 아니라 배선**이었습니다 — 모든 저장소에 복붙돼 있던 범용 페르소나
문서는 어떤 도구도 로드하지 않는 죽은 문서였습니다. 전부 걷어내고, 그 저장소에서 실제로 터졌던 사고를
점검 목록으로 갖는 서브에이전트와 사람이 건너뛸 수 없는 훅으로 대체했습니다.

### Sub-agent Topology

각 서브에이전트는 그 저장소에서 실제로 터졌던 장애 유형에 대응합니다.

```
                    ┌──────────────────────────────┐
                    │   ~/msa/AGENTS.md (Canon)     │
                    │   도구 무관 공통 규칙          │
                    │                               │
                    │   ┌─ Check & Claim            │
                    │   ├─ Worktree 세션 격리        │
                    │   ├─ Cross-Repo Impact Check  │
                    │   └─ GitHub Issues = SSOT     │
                    └──────────┬───────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
    ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │ Frontend     │  │ Backend          │  │ Infra            │
    ├──────────────┤  ├──────────────────┤  ├──────────────────┤
    │ ui-token-    │  │ flyway-migration-│  │ gateway-route-   │
    │  guard       │  │  guard           │  │  guard           │
    │  (토큰 미러   │  │  (엔티티↔마이그레 │  │  (로그인 전 경로  │
    │   불일치)     │  │   이션 누락)      │  │   화이트리스트)   │
    │              │  │                  │  │                  │
    │ shell-       │  │ tx-idempotency-  │  │ i18n / mermaid   │
    │  contract-   │  │  reviewer        │  │  무결성 훅        │
    │  guard       │  │  (재고 차감 멱등) │  │  (조용한 빈칸     │
    │  (v1 런타임   │  │                  │  │   렌더링)         │
    │   계약 파기)  │  │ cache-           │  │                  │
    │              │  │  invalidation-   │  │                  │
    │              │  │  guard           │  │                  │
    └──────────────┘  └──────────────────┘  └──────────────────┘

    공통: pre-push-verify 훅 — main push가 곧 프로덕션 배포인 저장소에서
          typecheck/test를 통과하지 못하면 push 도구 호출 자체를 차단
```

### Agent Governance Protocol

에이전트가 "지키기로 선언한 규칙"과 "기계가 실제로 강제하는 규칙"을 구분해 설계했습니다.

**기계가 강제하는 것 (hook / test — 건너뛸 수 없음)**

1. **Pre-push Verification** — CI 의존 금지. `main` push가 곧 프로덕션 배포인 구조라, 로컬
   `typecheck` / `test` 실패 시 push 도구 호출을 차단
2. **Design Token Mirror Check** — 손으로 유지되는 두 토큰 파일의 불일치와, 정의되지 않아
   조용히 죽는 CSS 변수 참조를 편집 직후 자동 검출
3. **i18n / Diagram Integrity Check** — 다국어 데이터에서 한 언어에만 없는 키(= 그 언어에서
   에러 없이 빈칸으로 렌더링)와 잘린 다이어그램 정의를 자동 검출
4. **Public Path Regression Test** — 로그인 전 접근해야 하는 경로를 테스트로 고정, 화이트리스트
   누락이 프로덕션이 아니라 빌드에서 실패하도록

**캐논이 규정하는 것 (`~/msa/AGENTS.md`)**

5. **Check & Claim + Worktree 격리** — 여러 AI 도구·세션을 동시에 운용하는 환경에서 같은 작업의
   중복 착수와 공용 클론의 물리적 충돌을 방지
6. **Cross-Repository Impact Analysis** — 공통 컴포넌트·API 스키마 변경 시 참조 저장소 사전 탐색.
   런타임 셸은 4개 프론트에 동시 반영되므로 특히 강제

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
- **Supply Chain Security** — Dependabot 의존성 자동 업데이트 + Trivy 컨테이너 이미지 취약점 스캔을 12개 저장소 전체에 롤아웃

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
