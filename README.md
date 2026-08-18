# 이도현 · Fullstack Architect & AI Agent Engineer

> **"복잡한 시스템을 AI 에이전트가 이해하고, 검수하고, 진화시킬 수 있도록 설계한다."**

MSA · Micro Frontends · AI Agent Orchestration을 관통하는 아키텍처를 설계하고,
셀프 호스팅 K3s 클러스터 위에서 서비스 메시부터 Observability까지 직접 운영합니다.

---

## Architecture Philosophy

- **End-to-End Ownership** — 도메인 모델링 → API 설계 → 프론트엔드 → 인프라 → 모니터링까지 수직 관통
- **AI-Native Development** — 코드를 작성하는 것이 아니라, 코드를 작성·검수·배포하는 **에이전트 시스템**을 설계
- **Design Decision as Code** — 모든 아키텍처 의사결정을 `AGENTS.md`, `task.md`, GitHub Projects로 코드화하여 추적

---

## 🏗️ PosSelect — Micro Frontends × MSA E-Commerce Platform

풀스택 마이크로서비스 및 마이크로 프론트엔드 아키텍처 기반의 커머스 플랫폼.
12개 독립 레포지토리가 런타임에 통합되며, 각 레포지토리는 AI 서브에이전트에 의해 자율적으로 검수됩니다.

### System Architecture

```
                        ┌─────────────────────────────────────────┐
                        │          posselect-shell (MFE Host)     │
                        │   Runtime Federation / Module Stitching │
                        └────┬──────┬──────┬──────┬──────┬───────┘
                             │      │      │      │      │
                    ┌────────┘  ┌───┘  ┌───┘  ┌───┘  ┌───┘
                    ▼           ▼      ▼      ▼      ▼
              customer.    product. admin.  store.  home.
               front       front   front   front   front
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
               │  Ingress + Service  │
               │  Mesh Routing       │
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

단순히 AI를 "사용"하는 것이 아니라, **11개 레포지토리 각각에 특화된 서브에이전트 페르소나를 설계**하고
이들이 코드 품질·아키텍처 정합성·배포 안전성을 자율적으로 검수하는 **에이전트 하네스 시스템**을 구축했습니다.

### Sub-agent Topology

```
                    ┌──────────────────────────────┐
                    │   Orchestrator Agent          │
                    │   (AGENTS.md Canon Rules)     │
                    │                               │
                    │   ┌─ Cross-Repo Impact Check  │
                    │   ├─ Rollback Plan Mandate    │
                    │   ├─ Pre-push Test Gate       │
                    │   └─ KI Pattern Compliance    │
                    └──────────┬───────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
    ┌─────▼──────┐   ┌────────▼───────┐   ┌───────▼────────┐
    │ Frontend   │   │   Backend      │   │  Infra         │
    │ Agents     │   │   Agents       │   │  Agents        │
    │            │   │                │   │                │
    │ shell      │   │ auth.api       │   │ architecture   │
    │ customer   │   │ product.api    │   │ gateway        │
    │ product    │   │ order.api      │   │                │
    │ admin      │   │                │   │                │
    │ store      │   │                │   │                │
    │ ui         │   │                │   │                │
    └────────────┘   └────────────────┘   └────────────────┘
```

### Agent Governance Protocol

각 서브에이전트는 다음 **6가지 거버넌스 원칙**을 강제합니다:

1. **GitHub Projects 일정 추적** — 모든 작업을 프로젝트 보드에 등록, 예상 일정(Milestone) 필수 기입
2. **Cross-Repository Impact Analysis** — 공통 컴포넌트·API 스키마 변경 시 전 레포지토리 영향도 사전 탐색
3. **Rollback Strategy Mandate** — 대규모 변경·배포 전 롤백 플랜 문서화 의무
4. **Pre-push Local Verification** — CI 의존 금지, `typecheck` / `test` 로컬 통과 후 Push
5. **Edge Case Coverage** — Happy Path 외 Timeout·404/500·Empty State 등 최소 3종 예외 처리 검증
6. **Knowledge Item Compliance** — 기존 아키텍처 패턴·코드 컨벤션과의 정합성 사전 검증

---

## 🔭 Observability & Infrastructure

### Self-hosted K3s Cluster (`leedohyun.com`)

프로덕션 워크로드를 운영하는 셀프 호스팅 Kubernetes 클러스터.
모니터링·보안·DNS까지 외부 매니지드 서비스 없이 직접 설계·운영합니다.

```
┌─ Observability ──────────────────────────────────┐
│  Grafana ◄── Prometheus/Alertmanager             │
│  Jaeger  ◄── OpenTelemetry Collector             │
│  Kiali   ◄── Istio Service Mesh Telemetry        │
└──────────────────────────────────────────────────┘

┌─ Data Layer ─────────────────────────────────────┐
│  PostgreSQL (×3)  │  MySQL  │  Redis (×2)        │
│  MinIO (S3)       │  imgproxy (실시간 리사이징)    │
└──────────────────────────────────────────────────┘

┌─ Security & Networking ──────────────────────────┐
│  Traefik Ingress  │  Istio Service Mesh          │
│  Keycloak SSO     │  Velero Backup               │
│  Route 53 DNS     │  SPF -all / DMARC reject     │
│  DDNS (동적 IP)    │  WAN 포트 최소화 (HTTP/S/SSH) │
└──────────────────────────────────────────────────┘
```

### Network Security Architecture

- **Attack Surface Minimization** — WAN 노출 포트를 HTTP/HTTPS/SSH로 한정, 불필요한 공격 벡터 제거
- **DNS Spoofing Prevention** — SPF `-all` + DMARC `reject` 정책으로 이메일 스푸핑 원천 차단
- **Dynamic IP Resilience** — DDNS 기반 도메인 운영으로 가정용 네트워크 환경에서도 안정적 서비스 제공

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
![Istio](https://img.shields.io/badge/Istio-466BB0?logo=istio&logoColor=white)
![Traefik](https://img.shields.io/badge/Traefik-24A1C1?logo=traefikproxy&logoColor=white)

**Observability** &nbsp;
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)
![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-000?logo=opentelemetry&logoColor=white)
![Jaeger](https://img.shields.io/badge/Jaeger-66CFE3?logo=jaeger&logoColor=000)

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
