# Security Compliance Admin
보안성 심사 대응용 어드민 시스템  
(인증/세션/2FA, RBAC, 감사로그, API Key, 다운로드 관리)

---

## 1. 프로젝트 개요
- **목적**: 보안 심사 기준을 충족하는 관리 콘솔을 2개월 내 개발하는 개인 프로젝트  
- **특징**: 인증/세션/비밀번호 정책, RBAC/ABAC 권한 제어, 감사 로그, API Key, 다운로드 관리 등 보안 요구사항 반영  

---

## 2. 주요 기능
- ✅ 인증/세션 관리 (ID/PW + TOTP 2FA, Idle 30분 자동 로그아웃)  
- ✅ 비밀번호 정책 (Argon2id, 최근 5개 재사용 금지)  
- ✅ RBAC/ABAC 기반 권한 제어  
- ✅ 감사 로그 저장·검색·CSV 내보내기  
- ✅ API Key 발급·회전·폐기  
- ✅ 파일 다운로드 권한 및 이력 관리  

---

## 3. 기술 스택
- **Frontend**: Thymeleaf + htmx + Bootstrap  
- **Backend**: Spring Boot 3, JDK 21  
- **Database**: PostgreSQL  
- **Cache/Session**: Redis  
- **Infra**: Docker Compose (Postgres, Redis)  

---

## 4. 프로젝트 구조
```plaintext
docs/               # 산출물 (요구사항, 아키텍처, ERD, 유스케이스 등)
src/                # Spring Boot 소스
infra/              # docker-compose 및 로컬 인프라
openapi/            # API 스펙
README.md
```

## 5. 실행 방법 (로컬 환경)

### 사전 요구사항
- JDK 21
- Docker & Docker Compose

### 실행
```bash
# 인프라 실행 (명령어 예정)

# 서버 실행 (명령어 예정)

```
### 접속: [http://localhost:8080](http://localhost:8080)


## 6. 문서 / 참고 자료
- 요구사항: [docs/01_requirements.md](docs/01_requirements.md)  
- 유스케이스: [docs/02_use_cases.md](docs/02_use_cases.md)  

## 7. 사용 목적
- 개인 포트폴리오
- 실제 운영환경 사용 목적 아님  
