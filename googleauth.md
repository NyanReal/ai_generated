# 단일 Google 로그인(SSO) + 경로 기반 라우팅 아키텍처

**대상 스택: Vue 프런트엔드, ASP.NET Core 백엔드, MariaDB (도커 분리 배치)**

여기서는 **한 대의 머신**에서 **Nginx 리버스 프록시**와 **Let’s Encrypt TLS**를 사용해 **경로 기반 라우팅**을 하고, **Google OIDC 단일 로그인(SSO)** 으로 모든 서비스를 인증 상태로 만드는 전체 구조와 요청 흐름을 설명합니다. 각 서비스는 Vue SPA(정적) + ASP.NET Core API로 구성되고, 영속 저장은 MariaDB를 사용합니다. 모든 컴포넌트는 역할별로 도커 컨테이너로 분리합니다.

---

## 전체 구조(High-Level Architecture)

```
Internet
   │   https://your.domain/
   ▼
[Edge Nginx + Let's Encrypt]  ← certbot/lego
   │  (TLS 종단, 경로 라우팅, auth_request로 인증 게이트)
   │      └─ auth_request → [oauth2-proxy (Google OIDC)]
   │                         │
   │     (세션 유효 → 통과, 아니면 Google로 리다이렉트)
   ▼
 ┌──────────────────────────────────────────────────────────────┐
 │                      Internal Docker Network                 │
 │                                                              │
 │  [frontend-a]   [frontend-b]   [frontend-c]   (Vue 정적)     │
 │      │              │              │                          │
 │    /a/*           /b/*           /c/*                         │
 │      ▼              ▼              ▼                          │
 │   [api-a]        [api-b]        [api-c]        (ASP.NET Core) │
 │      │              │              │                          │
 │  MariaDB-a      MariaDB-b     MariaDB (공유 또는 서비스별)     │
 │                                                              │
 │  (선택) [Redis]  ← oauth2-proxy 세션 저장소                   │
 └──────────────────────────────────────────────────────────────┘
```

---

## 주요 컴포넌트와 역할

* **Edge Nginx**: 유일한 공개 엔드포인트. TLS 종단, URL 경로 기반 라우팅, `auth_request`로 로그인 강제.
* **oauth2-proxy**: Google OIDC 로그인 처리·검증, **도메인 공용 세션 쿠키** 발급(SSO 실현).
* **Vue 프런트엔드**: 빌드된 정적 자산(컨테이너 내 경량 Nginx 등으로 서빙).
* **ASP.NET Core 백엔드**: 인증된 요청만 수신. Edge Nginx가 주입한 신뢰 가능한 헤더를 사용하거나 Bearer 토큰 검증.
* **MariaDB**: 단일 인스턴스에 서비스별 스키마·계정 분리 또는 인스턴스 분리. 최소권한과 백업 전략 적용.
* **Redis(선택)**: oauth2-proxy 세션 서버측 저장(짧은 쿠키, 서버측 무효화가 필요할 때).
* **certbot/lego**: 자동 TLS 인증서 발급/갱신.

---

## 경로 설계 예시

* **/a/** → frontend-a (정적 SPA)
* **/a/api/** → api-a (ASP.NET Core)
* **/b/** → frontend-b (정적 SPA)
* **/b/api/** → api-b (ASP.NET Core)
* **/oauth2/** → oauth2-proxy (로그인, 콜백, 로그아웃)

> 모든 경로가 **같은 상위 도메인**(예: `your.domain`) 아래에 존재하므로, **세션 쿠키 1개**로 A/B/C 전체가 **SSO** 상태를 공유합니다.

---

## SSO 요청 흐름

1. 사용자가 \*\*/a/\*\*에 접근
2. Edge Nginx가 **/oauth2/auth**(oauth2-proxy)로 `auth_request` 서브요청을 보내 **세션 유효성** 확인
3. 세션이 없으면 oauth2-proxy가 **Google 로그인**으로 리다이렉트 → 성공 시 **/oauth2/callback**으로 돌아와 **도메인 공용 세션 쿠키**(예: `_oauth2_proxy`) 발급
4. 이후 정적 리소스 응답
5. **/a/api/** 요청도 동일하게 `auth_request` 거치며, Nginx가 **X-Auth-Request-Email** 등 **신원 헤더**(또는 Authorization 헤더)를 백엔드에 주입

---

## 보안 경계 & 베스트 프랙티스

* 백엔드는 **공개하지 않고 내부 네트워크 바인딩**만. 외부 공개는 **Edge Nginx**뿐.
* 클라이언트가 보낸 **임의의 `X-Auth-*` / `Authorization` 헤더는 제거**하고, **Edge**가 **직접 설정한 헤더만** 업스트림에 주입.
* `auth_request`가 200일 때만 업스트림 프록시. 미인증은 401 → `/oauth2/start`로 로그인 유도.
* 세션 쿠키는 **Secure/HttpOnly/SameSite=Lax** 권장.
* 필요 시 **CSP**, **Rate Limit**, **(경량) WAF** 추가.

---

## CORS & CSRF

* 전부 **same-origin** 구성이므로 **CORS는 보통 불필요**.
* CSRF는 **SameSite=Lax** 쿠키에 더해, **상태 변경 API**에 **CSRF 토큰**(헤더/더블 서브밋 쿠키)을 사용해 보강.

---

## WebSocket

* Nginx에서 **프로토콜 업그레이드** 처리가 필요.
* `auth_request`는 업그레이드 연결에는 지속 적용되지 않으므로, **HTTP에서 미리 인증(세션 쿠키 또는 단기 JWT)** → **웹소켓 핸드셰이크 시 쿠키/JWT 검증**(예: `Sec-WebSocket-Protocol` 또는 커스텀 헤더) 패턴을 권장.

---

## MariaDB 분리 전략

* 가장 단순: **단일 인스턴스 + 서비스별 스키마**(예: `img_db`, `yt_db`)
* **스키마별 전용 DB 계정**으로 최소 권한.
* **정기 백업**(스키마 단위 덤프 또는 인스턴스 스냅샷).

---

## 컨테이너 구성(최소 세트)

* **edge-nginx**: 리버스 프록시, TLS 종단, 경로 라우팅, `auth_request` 게이트
* **oauth2-proxy**: Google OIDC 클라이언트, 세션 발급/검증
* **frontend-a/b/c**: Vue 정적 파일 서빙(Nginx-alpine)
* **api-a/b/c**: ASP.NET Core(Kestrel)
* **mariadb**: 단일 인스턴스(스키마 분리) 또는 서비스별 인스턴스
* **certbot/lego**: 인증서 자동 발급/갱신
* **redis(선택)**: oauth2-proxy 세션 저장

---

## Nginx 개념 스니펫(`auth_request` + 경로 라우팅)

```nginx
server {
  listen 443 ssl http2;
  server_name your.domain;

  ## TLS & ACME 설정 …

  ## 인증 체크(서브요청)
  location = /oauth2/auth {
    internal;
    proxy_pass        http://oauth2-proxy:4180/oauth2/auth;
    proxy_set_header  X-Original-URI $request_uri;
    proxy_set_header  X-Real-IP      $remote_addr;
    proxy_set_header  X-Auth-Request-Redirect $scheme://$host$request_uri;
  }

  ## oauth2-proxy 엔드포인트(로그인/콜백/로그아웃)
  location /oauth2/ {
    proxy_pass http://oauth2-proxy:4180;
  }

  ## 서비스 A 정적
  location ^~ /a/ {
    auth_request /oauth2/auth;
    error_page 401 = /oauth2/start?rd=$scheme://$host$request_uri;

    # Edge가 설정한 신원 값 전달(신뢰 경계 내부)
    auth_request_set $user  $upstream_http_x_auth_request_user;
    auth_request_set $email $upstream_http_x_auth_request_email;

    proxy_set_header X-User  $user;
    proxy_set_header X-Email $email;
    proxy_pass http://frontend-a;
  }

  ## 서비스 A API
  location ^~ /a/api/ {
    auth_request /oauth2/auth;
    error_page 401 = /oauth2/start?rd=$scheme://$host$request_uri;

    # 클라이언트 Authorization 헤더 스푸핑 차단
    proxy_set_header Authorization "";

    # 필요 시 oauth2-proxy가 내려준 신원 헤더/Bearer 전달
    proxy_set_header X-User  $upstream_http_x_auth_request_user;
    proxy_set_header X-Email $upstream_http_x_auth_request_email;

    proxy_pass http://api-a;
  }

  ## /b/, /b/api/, /c/, /c/api/ 도 동일 패턴…
}
```

---

## oauth2-proxy 핵심 옵션

* `--provider=google`
* `--client-id=…  --client-secret=…`
* `--redirect-url=https://your.domain/oauth2/callback`
* `--cookie-domain=your.domain  --cookie-secure=true  --cookie-samesite=lax`
* *(선택)* `--session-store-type=redis  --redis-connection-url=…`
* *(선택)* `--set-authorization-header=true`  (백엔드가 Bearer 토큰 선호 시)

---

## ASP.NET Core에서의 사용자 식별

두 가지 일반 패턴:

* **헤더 기반**: Edge Nginx가 주입한 **신뢰 헤더**(예: `X-Email`)를 읽어 현재 사용자 컨텍스트에 매핑.

  * 백엔드는 반드시 리버스 프록시 뒤에서만 노출되고, **필수 헤더 없으면 401** 처리.
* **토큰 기반**: oauth2-proxy가 `Authorization: Bearer …`를 세팅하도록 하고, 백엔드에서 **JWT 검증**(발행자/공개키) 수행.

  * 서비스 간 호출, 세분화된 권한 스코프 관리에 유리.

---

## 대안 아키텍처(검토 시점)

* **각 ASP.NET Core 앱이 직접 OIDC 적용**: 앱별 세밀 제어 가능하지만 공유 데이터 프로텍션 키, 쿠키 도메인/경로, 리디렉트 등 **운영 포인트 증가**.
* **내부 IdP(Keycloak/Authentik) + Google 연동**: 강력한 RBAC, 중앙화된 권한 관리. 단, **단일 머신/경량 구성에는 과함**.

---

## 메모(Notes)

* **하나의 상위 도메인** 아래에 모두 배치해 SSO 쿠키를 **공유**하세요.
* DB는 **서비스별 스키마/계정 분리** + 최소 권한 원칙을 지키세요.
* WebSocket은 **HTTP 선인증 + 핸드셰이크 시 쿠키/JWT 검증** 패턴이 안정적입니다.
