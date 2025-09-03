# Security Compliance Admin — 요구사항 명세서 (v0.1)
보안성 심사 대응용 어드민 시스템

## 목차
- [Security Compliance Admin — 요구사항 명세서 (v0.1)](#security-compliance-admin--요구사항-명세서-v01)
  - [목차](#목차)
  - [0. 문서 메타](#0-문서-메타)
  - [1. 개요](#1-개요)
    - [1.1 목적](#11-목적)
    - [1.2 범위](#12-범위)
      - [1.2.1 In-Scope (MVP)](#121-in-scope-mvp)
      - [1.2.2 Out-of-Scope (이번 릴리즈 제외)](#122-out-of-scope-이번-릴리즈-제외)
    - [1.3 이해관계자 \& 역할](#13-이해관계자--역할)
    - [1.4 가정/제약](#14-가정제약)
  - [2. 일반 요구사항](#2-일반-요구사항)
  - [3. 상세 요구사항](#3-상세-요구사항)
    - [3.1 기능 요구사항 (Functional Requirements, FR)](#31-기능-요구사항-functional-requirements-fr)
    - [3.2 비기능 요구사항 (Non-Functional, NFR)](#32-비기능-요구사항-non-functional-nfr)
    - [3.3 데이터 요구사항 (요약)](#33-데이터-요구사항-요약)
    - [3.4 오류/보안 이벤트](#34-오류보안-이벤트)
    - [3.5 추적성](#35-추적성)
  - [4. 용어 사전](#4-용어-사전)
  - [5. 참조 문서](#5-참조-문서)

## 0. 문서 메타
- 문서 버전: v0.1 (2025-09-02)
- 문서 소유자: 김지민
- 변경이력:
  - v0.1 (2025-09-02) 초안 작성

## 1. 개요
### 1.1 목적
- 목적: 보안 심사 대응이 가능한 Admin 시스템의 기능/비기능 요구 정의(Dashboard & REST API)
- GitHub Repo: [security-compliance-admin](https://github.com/zz-min/security-compliance-admin)

### 1.2 범위
#### 1.2.1 In-Scope (MVP)
   - 인증/세션/2FA: ID/PW + TOTP, Idle 30m 자동 로그아웃, 실패 5회 잠금, 90일 비사용 잠금, 90일 비밀번호 강제 변경
   - 비밀번호 정책: 8~20자/3종 조합, Argon2id 저장, 최근 5개 재사용 금지
   - RBAC(+ABAC): 4역할 + organization_id 범위 제한, 권한 변경 감사
   - 감사/트래킹 로그: 핵심 이벤트 기록·검색·CSV
   - API Key: 발급/회전/폐기, 사용 이력(최소 최종 사용시각/엔드포인트), 마스킹, 해시 저장
   - 파일 다운로드 관리: 권한 체크 + 요청/실행 이력 기록

#### 1.2.2 Out-of-Scope (이번 릴리즈 제외)
- 절대(Absolute) 세션 타임아웃
- 타임아웃에 따른 자동 로그아웃 사전 경고 UX 팝업
- 감사 로그 무결성 해시체인 검증
- 파일 다운로드 승인 워크플로(요청→승인/거절)
- 레이트 리밋(일정 시간 동안 요청 횟수를 제한)
- 감사로그 보존/백업
- HTTPS
 
### 1.3 이해관계자 & 역할  
| 역할      | 책임/목표                 | 주요 권한(요약)      |
| --------- | ------------------------- | -------------------- |
| ADMIN     | 사용자/권한/정책, API Key | 사용자·역할 관리 등  |
| AUDITOR   | 감사 추적/검증            | 감사 로그 조회/CSV   |
| OPERATOR  | 운영 통계                 | 고객사 제한 뷰       |
| DEVELOPER | 데이터 다운로드           | 고객사 제한 다운로드 |

### 1.4 가정/제약
- **스택/런타임**: FE Thymeleaf + htmx + Bootstrap , BE Spring Boot 3, JDK 21, PostgreSQL , Redis
- **타임존/시간 처리**: 서버 저장은 **UTC**, UI 표시는 **Asia/Seoul(KST)**, 포맷은 **ISO-8601**
- **보안 기본**: 사내망 전용 콘솔 가정, 시크릿은 `.env.local`/KMS에 보관(레포 커밋 금지)
- **데이터 민감도**: PII/민감 데이터는 최소 수집·표시 시 마스킹
- **일정/리소스**: 1인, 2개월, 데모/포트폴리오 목적(운영 고가용성은 범위 외)
  
## 2. 일반 요구사항
> 시스템 전반에 공통으로 적용되는 정책/규칙

- **API 규약**
  - REST/JSON UTF-8, 기본 응답 컨벤션 권장:  
    `{ "data": <payload|null>, "error": { "code": "<ERR_CODE>", "message": "<human readable>", "traceId": "<uuid>" } }`
  - 표준 상태코드 사용: 200/201/204/400/401/403/404/409/422/500
  - **CORS**: 관리 콘솔 도메인 화이트리스트 기반 허용

- **보안 기본**
  - 세션 쿠키: `HttpOnly`, `Secure`, `SameSite=Strict`; 로그인/권한상승 시 **세션ID 재발급**
  - CSRF 보호: 콘솔 도메인 한정 + CSRF 토큰(비-API 폼) / API는 stateless + 헤더 토큰 검증
  - 입력 검증: 서버(Bean Validation) 필수, 클라이언트는 보조
  - 로깅: 시크릿/OTP/API Key **미로그**, PII 마스킹, `traceId` 포함

- **권한/데이터 경계**
  - 모든 보호 API는 **RBAC + orgId(ABAC)** 동시 적용
  - 기본 조회는 **자기 조직 범위**만 반환(ADMIN 예외)

- **페이지네이션/검색/내보내기**
  - 페이지: `page=0, size=20(≤100)`, 정렬: `sort=field,dir`
  - 내보내기: **CSV(Excel 호환, UTF-8)**
  - **파일명 규칙:** `{파일명}_{yyyyMMddHHmm}_KST.csv`

- **운영/관측성**
  - 로컬 실행: `infra/docker-compose.yml`(Postgres, Redis)
  - 헬스체크: `/actuator/health` 공개, 상세는 내부만
  - 로그 포맷: JSON 권장(레벨/traceId/actor/role/IP)

## 3. 상세 요구사항

### 3.1 기능 요구사항 (Functional Requirements, FR)

<a id="fr-001"></a>
<details>
  <summary><strong>FR-001 인증/2FA</strong> — <em>MUST</em></summary>

- 설명: ID/PW + TOTP 6자리 로그인
- 수용 기준 (GWT):
  - **Given** 등록 사용자, **When** 올바른 TOTP를 제출하면, **Then** 200과 함께 세션 생성
  - **Given** TOTP 또는 비번 5회 연속 실패, **When** 재시도, **Then** 계정 `LOCKED` 및 감사 이벤트 기록

</details>

<a id="fr-002"></a>
<details>
  <summary><strong>FR-002 세션/자동 로그아웃</strong> — <em>MUST</em></summary>

- 설명: Spring Session(+Redis), Idle 30분 만료
- 수용 기준:
  - 마지막 요청 후 30분 경과 시 401/세션 파기(`SESSION_EXPIRED`) 기록
  - 쿠키: `HttpOnly`, `Secure`, `SameSite=Strict`
  - 로그인/권한상승 시 세션ID 재발급(고정화 방지)
  - 세션 생성/재발급/파기 감사 이벤트 기록

</details>

<a id="fr-003"></a>
<details>
  <summary><strong>FR-003 비밀번호 정책</strong> — <em>MUST</em></summary>

- 설명: 8~20자, 대/소/숫자/특수 중 3종 이상, Argon2id 저장, 최근 5개 재사용 금지
- 수용 기준:
  - Argon2id 파라미터 `t≥3, m≥32MiB, p≥1` (솔트 자동/내장)
  - 변경 시 `password_history` 최근 5개 해시와 비교해 재사용 차단

</details>

<a id="fr-004"></a>
<details>
  <summary><strong>FR-004 비번/계정 주기 정책</strong> — <em>MUST</em></summary>

- 설명: 90일 비번 강제 변경, 90일 미사용 계정 자동 잠금
- 수용 기준:
  - 로그인 성공 시 `password_changed_at` 90일 초과면 변경 플로우 진입
  - 매일 00:00 스케줄러가 `last_login_at` 90일 초과 계정을 `LOCKED`로 전환

</details>

<a id="fr-020"></a>
<details>
  <summary><strong>FR-020 RBAC(+ABAC)</strong> — <em>MUST</em></summary>

- 설명: ADMIN/AUDITOR/OPERATOR/DEVELOPER 역할 + organization 범위 제한
- 수용 기준:
  - 모든 보호 API는 역할 권한 검사 + `organization_id` 가드 적용
  - 역할/권한 생성·수정·삭제는 감사 로그 기록

</details>

<a id="fr-030"></a>
<details>
  <summary><strong>FR-030 감사/트래킹 로그</strong> — <em>MUST</em></summary>

- 설명: 주요 인증/권한/데이터/운영 이벤트를 저장·검색·내보내기
- 수용 기준:
  - 필터: 기간 / 사용자 / 역할 / 행위 / 결과
  - CSV 내보내기 전 권한 체크(AUDITOR/ADMIN)
  - 기록 이벤트 범주:
    - **인증/세션**
      - 로그인/로그아웃
      - 로그인 실패(비밀번호/TOTP)
      - 계정 잠금/해제
      - 세션 만료/강제 로그아웃
      - 2FA 등록/해제
    - **계정/권한**
      - 계정 생성/삭제
      - 계정 정보 변경 (이메일, 전화번호, 이름, 소속 등)
      - 비밀번호 변경/초기화
      - 권한(Role/Permission) 생성/수정/삭제
    - **API Key**
      - 발급/회전/폐기
      - 사용 성공/실패 (만료, 권한 부족 포함)
    - **데이터 접근**
      - 민감 데이터 조회
      - 파일 다운로드 요청/실행/거부
    - **보안 이벤트(실패)**
      - 권한 없는 메뉴/API 접근 (403)
      - 기타 정책 위반 (422) 기록
</details>

<a id="fr-040"></a>
<details>
  <summary><strong>FR-040 API Key 라이프사이클</strong> — <em>MUST</em></summary>

- 설명: 발급/회전/폐기, 사용 이력(최소 최종 사용시각/엔드포인트)
- 수용 기준:
  - 발급 직후 1회만 전체 노출, 이후 prefix+마스킹
  - 저장은 해시(접두 태그 규칙 포함), 이벤트 로그에 엔드포인트/결과 기록

</details>

<a id="fr-050"></a>
<details>
  <summary><strong>FR-050 파일 다운로드 관리</strong> — <em>MUST</em></summary>

- 설명: 민감 데이터 다운로드 권한 체크 및 요청/실행 이력 기록
- 수용 기준:
  - 무권한 접근은 403 및 `DOWNLOAD_DENIED` 감사 이벤트 기록

</details>


### 3.2 비기능 요구사항 (Non-Functional, NFR)

- **NFR-SEC(보안)**: CSRF 방어, 최소권한, 세션 하드닝, 감사 로깅, 비밀번호 Argon2id
- **NFR-PERF(성능)**: 로그인/비번 변경 API p95 < 500ms, 해시 100~300ms 목표
- **NFR-OPS(운영)**: 로컬 `docker-compose`(Postgres, Redis), 로그 레벨/PII 마스킹
- **NFR-API(호환)**: OpenAPI 3.0 문서 제공(springdoc), CORS 설정

### 3.3 데이터 요구사항 (요약)
- 주요 엔터티: `users`, `password_history`, `roles/permissions/user_roles`,
  `audit_logs`, `api_keys/api_key_events`, `file_resources`, `download_requests`, `download_events`
- 인덱스 권장: `audit_logs(ts)`, `api_key_events(api_key_id, ts)`, `download_requests(status, created_at)`
- 개인정보/민감 데이터: 최소 수집/마스킹, 키/시크릿/복구코드는 암호화 저장
  
### 3.4 오류/보안 이벤트
- **상태코드 규칙**
  - **401**: 미인증/세션 만료(Idle)  
  - **403**: 권한 부족/RBAC·ABAC 위배  
  - **409**: 자원 상태 충돌(예: 키 회전 조건 불일치)  
  - **422**: 정책 위반(비밀번호 재사용 등)  
  - **423**: 계정 잠금(로그인 실패 5회) — 선택 사용  

- **에러 응답 포맷(권장)**
  ```json
  {
    "data": null,
    "error": {
      "code": "AUTH_TOTP_INVALID",
      "message": "Invalid TOTP code.",
      "traceId": "8c9b1f4e-9a2f-4f0f-a9a1-3f6c5a7d1e2b"
    }
  }
  ```

### 3.5 추적성
| FR-ID                 | 관련 UC | 비고                       |
| --------------------- | ------- | -------------------------- |
| [FR-001](#fr-001)     | UC-001  | 로그인+2FA/실패잠금        |
| [FR-002](#fr-002)     | UC-002  | Idle 30m 만료              |
| [FR-003/004](#fr-003) | UC-003  | 비번 변경/재사용 금지/주기 |
| [FR-030](#fr-030)     | UC-004  | 감사 조회/CSV              |
| [FR-040](#fr-040)     | UC-005  | API Key 발급/회전/폐기     |
| [FR-050](#fr-050)     | UC-006  | 다운로드 권한/이력         |

## 4. 용어 사전
| 용어/약어                                               | 정의                                                                                   |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| 2FA (Two-Factor Authentication)                         | 사용자 인증 시 ID/PW 외에 OTP 같은 추가 인증 요소를 요구하는 방식                      |
| TOTP (Time-based One-Time Password)                     | Google Authenticator 등에서 사용하는 30초 주기의 일회용 비밀번호 알고리즘 (RFC 6238)   |
| RBAC (Role-Based Access Control)                        | 역할(Role)에 따라 접근 권한을 부여하는 방식 (예: ADMIN, AUDITOR, OPERATOR, DEVELOPER)  |
| ABAC (Attribute-Based Access Control)                   | 속성(Attribute, 예: organization_id)에 기반해 접근을 제어하는 방식                     |
| Idle Timeout                                            | 사용자의 마지막 활동 후 일정 시간이 지나면 세션이 만료되는 정책 (MVP: 30분)            |
| Argon2id                                                | 최신 권장 비밀번호 해싱 알고리즘 (메모리 집약적, GPU 공격 방어)                        |
| Audit Log (감사 로그)                                   | 시스템에서 발생한 주요 보안 이벤트(로그인, 계정 잠금, 권한 변경 등)를 추적/기록한 로그 |
| CSV (Comma-Separated Values)                            | 표 형식 데이터를 쉼표로 구분하여 저장하는 파일 형식. Excel 호환                        |
| Organization (조직/고객사)                              | 사용자 계정이 속한 사업 단위. RBAC/ABAC와 함께 API 범위 제한에 활용됨                  |
| PII (Personally Identifiable Information, 개인식별정보) | 사람을 직접 식별할 수 있는 모든 데이터                                                 |

## 5. 참조 문서
- **02_use_cases.md** — 유스케이스 상세 (작성 예정)
- **03_architecture.md** — 시스템 아키텍처 및 설계 (작성 예정)
- **04_db_erd.md** — 데이터 모델/ERD (작성 예정)
