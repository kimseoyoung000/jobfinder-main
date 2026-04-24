# JobFinder — 개발자 맞춤형 구인구직 플랫폼 (메인서비스)

> Spring Boot 기반 MSA 아키텍처 구인구직 플랫폼  
> **담당 파트: 구직자(Jobseeker) 기능 전체 + DB 설계**

## 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 개발 기간 | 2026.02.24 ~ 2026.03.31 (약 5주) |
| 인원 | 4명 |
| 담당 | 구직자 기능 전체 (9개 기능 영역) |
| GitHub | [jobfinder-main](https://github.com/kimseoyoung000/jobfinder-main) / [jobfinder-board](https://github.com/kimseoyoung000/jobfinder-board) / [jobfinder-gateway](https://github.com/kimseoyoung000/jobfinder-gateway) / [jobfinder-discovery](https://github.com/kimseoyoung000/jobfinder-discovery) |

## 기술 스택

| 구분 | 기술 |
|------|------|
| Backend | Java 17, Spring Boot, Spring Security, MyBatis, Oracle DB |
| Frontend | Thymeleaf, HTML5, CSS, JavaScript, jQuery |
| Auth | JWT (Access Token / Refresh Token) |
| MSA | Spring Cloud Gateway, Eureka Discovery, LoadBalancer |
| Infra | AWS EC2, Docker, Docker Hub, Jenkins CI/CD |
| Quality | SonarQube, Git/GitHub |

## 시스템 아키텍처

```
Client (Browser)
    ↓
API Gateway (8000) — 단일 진입점, JWT 검증, 서비스 라우팅
    ↓
Eureka Discovery (8761) — 서비스 등록/조회
    ↓                ↓
User Service (8001)   Board Service (8002)
(메인서비스)           (게시판서비스)
    ↓
Oracle Database
```

- Gateway가 단일 진입점으로 요청 수신 → Eureka로 서비스 조회 후 URL 기반 라우팅
- 각 서비스는 독립 동작, JWT 인증은 Gateway에 통합
- Docker + Jenkins CI/CD로 자동 빌드/배포

## 담당 기능

### 마이페이지 대시보드
- 프로필 카드 (희망직무/희망지역 태그 → 추천 공고 매칭 기준)
- 활동 통계 4종 (지원내역/받은제안/스크랩공고/관심기업 실시간 집계)
- 채용 캘린더 (FullCalendar — 지원/마감/제안기한 색상 구분)
- 맞춤 추천 공고 (대표이력서 분석 → 매칭률 % 스코어링)

### 이력서 관리 CRUD
- 완성도 프로그레스바 (단계별 색상 구분)
- 대표 이력서 1건 지정 (clearPrimary → setPrimary 순서로 중복 방지)
- 프로필 사진 AJAX 업로드/미리보기
- 지원 시 JSON → Oracle CLOB 스냅샷 저장

### 채용공고 목록/상세
- 다중 조건 필터 (지역/경력/학력/직무/기술스택 등 8종)
- 이력서 기반 매칭도 계산 (지역 + 직무 + 연봉 + 기술스택 가중 점수)
- 지원자 통계 (성별/연령/학력/경력/자격증 Top5)
- 비슷한 공고 추천 (유사도 % 기반)

### 입사지원 관리
- 중복 지원 방지, 드래그앤드롭 첨부 (PDF/DOC/PPT/ZIP, 최대 5개)
- 6종 상태별 탭 필터 + 키워드 검색
- 5단계 상태 통계 카드 (지원/심사/면접/합격/불합격)
- 지원 취소 (지원완료 + 기업 미열람 상태만 허용)

### 받은 제안 관리
- 4단계 상태 통계 카드 (미열람/검토중/수락/거절)
- 미열람 카드 강조 UI + 클릭 시 열람 처리
- 수락/거절 → OfferService 상태 변경 + 기업 알림

### 기업정보 조회
- 다중 조건 필터 (업종/기업규모/지역 + 채용중)
- 유사 기업 추천 (업종 기반 최대 5개)
- 팔로잉 원클릭 AJAX

### 스크랩 / 최근 본 공고 / 팔로우 기업
- 날짜별 자동 그룹 + 기간 필터
- AJAX 즉시 처리 + Oracle ROWNUM 기반 페이지네이션

### DB 설계 참여
- 45개 테이블, 30개 시퀀스 — 통합 DDL 작성
- 회원/기업/채용공고/이력서/입사지원 5개 핵심 도메인 정규화 설계

## 핵심 설계 포인트

### 1. 이력서 스냅샷 (JSON → CLOB)
지원 시점의 이력서를 `ObjectMapper.writeValueAsString()`으로 JSON 변환 후 Oracle CLOB 컬럼에 영구 보존. 회계 결산 마감 후 원장이 수정되지 않는 원리를 동일하게 적용하여 과거 지원 내역의 데이터 무결성을 확보했습니다.

- `jdbcType=CLOB`을 명시하지 않으면 VARCHAR2(4000byte)로 자동 처리돼 ORA-01461 에러 발생
- 지원 후 이력서를 수정해도 기업이 보는 지원서는 그대로 유지

### 2. 이력서 기반 매칭도 계산
공고의 요구 기술스택과 구직자 이력서의 기술을 비교해 점수화. 지역(5점) + 직무(3점) + 연봉(2점) + 기술스택(요구 수 × 2점)을 만점 대비 퍼센트로 환산하는 가변 가중치 구조입니다.

### 3. MSA 4개 서비스 도메인 분리
Discovery / Gateway / Main / Board로 책임 분리. JWT 인증은 Gateway에 통합해 각 서비스는 비즈니스 로직에 집중하도록 구성했습니다.

## 트러블슈팅

### MSA 전환 시 URL prefix 404 문제
- **문제:** Gateway 경유 시 redirect URL에 prefix 누락 → 404 에러
- **원인:** Gateway가 prefix 제거 후 전달하지만 서비스 내부 redirect 시 prefix 미포함
- **해결:** `application.yml`에 `forward-headers-strategy: framework` 설정 추가

### 이력서 스냅샷 CLOB 저장 문제
- **문제:** JSON → CLOB 저장 시 4000byte 초과에서 ORA-01461 에러
- **원인:** MyBatis에서 타입 미지정 시 VARCHAR2(최대 4000byte)로 자동 처리
- **해결:** MyBatis XML에서 `jdbcType=CLOB` 명시

### 대표 이력서 중복 방지
- **문제:** 대표 이력서 설정 시 기존 대표가 해제되지 않는 버그
- **해결:** `clearPrimaryResume`(전체 해제) → `setPrimaryResume`(해당 건 설정) 순서로 호출

## CI/CD 배포 흐름

```
GitHub Push → Jenkins (Webhook 트리거, Gradle 빌드)
→ Docker 이미지 빌드 → Docker Hub Push
→ AWS EC2 Pull & Run (docker-compose)
```

- SonarQube 정적 분석으로 코드 품질 관리
- Local VM(테스트) + AWS EC2(운영) 이중 환경

## 환경변수 설정

민감 정보는 환경변수로 관리합니다. 실행 시 아래 환경변수를 설정해주세요.

```
JWT_SECRET=your-jwt-secret-key
DB_URL=jdbc:oracle:thin:@your-db-host:1521:xe
DB_USERNAME=your-db-username
DB_PASSWORD=your-db-password
SOLAPI_API_KEY=your-solapi-api-key
SOLAPI_API_SECRET=your-solapi-api-secret
SOLAPI_FROM_PHONE=your-phone-number
PORTONE_IMP_CODE=your-imp-code
PORTONE_API_KEY=your-portone-api-key
PORTONE_API_SECRET=your-portone-api-secret
```
