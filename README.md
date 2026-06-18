# Flare On The Road

Flare On The Road는 공공 ITS CCTV 영상을 기반으로 도로 위 화재 및 연기 위험 징후를 탐지하고, 탐지 이벤트를 대시보드에서 확인할 수 있도록 구성한 AI 관제 서비스입니다. 단순히 모델을 실행하는 데서 끝내지 않고, 프론트엔드, 백엔드, 비전 모델 서버, 수집 워커를 분리해 실제 서비스 운영 흐름에 가깝게 설계했습니다.

이 프로젝트는 실무 환경에서 자주 마주치는 요구사항인 인증, 게시판, 관리자 기능, 파일 업로드, 외부 API 연동, AI 추론 서버 연동, 이벤트 저장 및 조회, 프론트엔드 상태 관리까지 하나의 서비스 흐름으로 연결한 것이 특징입니다.

## 핵심 문제 정의

고속도로 및 터널 CCTV는 실시간성이 중요하지만 사람이 모든 영상을 지속적으로 확인하기 어렵습니다. Flare는 CCTV 프레임을 주기적으로 수집하고, 1차 객체 탐지 모델로 `fire`, `smoke`, `carlight` 후보를 찾은 뒤, 위험 이벤트를 백엔드에 저장하여 관제 화면에서 확인할 수 있도록 합니다.

주요 목표는 다음과 같습니다.

- 공공 CCTV API를 활용한 실시간 영상 목록 및 스트림 확인
- RT-DETRv2 기반 화재/연기 후보 탐지
- Worker를 통한 CCTV 프레임 수집, 이미지 저장, AI 추론 호출, 이벤트 적재
- Flask 백엔드를 통한 인증, 도메인 API, 이벤트 조회, AI Lab API 제공
- Next.js 프론트엔드를 통한 관제 대시보드, 이벤트 목록, AI 모델 데모 제공
- 계층형 백엔드 구조와 타입 기반 프론트엔드 구조로 유지보수성 확보

## 전체 아키텍처

```text
사용자
  |
  v
Next.js Frontend
  - App Router 기반 화면 구성
  - API proxy route
  - Zustand 인증 상태 관리
  - CCTV 지도/영상/이벤트/AI Lab UI
  |
  v
Flask Backend API
  - Route / Service / Repository / Model 계층
  - JWT 인증 및 OAuth
  - 게시판, 댓글, 관리자, 파일, 이벤트 API
  - Vision API 연동용 AI Lab API
  |
  +------------------------+
  |                        |
  v                        v
Database              Cloudflare R2
  - users/posts       - CCTV snapshot image
  - comments/events   - presigned URL 기반 조회

별도 런타임
  |
  +--> Python Worker
  |      - ITS CCTV API 호출
  |      - ffmpeg/OpenCV 프레임 캡처
  |      - Vision API 호출
  |      - 이벤트를 Backend API로 저장
  |
  +--> Vision Model Server
         - FastAPI
         - RT-DETRv2 ONNX
         - fire/smoke/carlight 탐지 결과 반환
```

프론트엔드, 백엔드, 워커, 모델 서버를 분리한 이유는 각 영역의 변경 주기와 장애 범위가 다르기 때문입니다. UI 변경은 프론트엔드에서 독립적으로 진행하고, API 도메인 로직은 백엔드에서 관리하며, 무거운 AI 추론과 CCTV 수집 작업은 별도 프로세스로 격리했습니다. 이 구조는 실무에서 흔히 사용하는 서비스 분리, 외부 연동 격리, 장애 전파 최소화 관점과 맞닿아 있습니다.

## 백엔드 구조

백엔드는 Flask 기반이며 `app` 패키지 안에서 역할별 계층을 분리했습니다.

```text
backend/
  app/
    routes/        HTTP endpoint와 request/response 경계
    services/      비즈니스 로직과 외부 API 연동
    repositories/  DB 접근 로직
    models/        SQLAlchemy ORM 모델
    schemas/       요청/응답 스키마 및 직렬화 보조
    common/        공통 응답, 예외, 업로드, 데코레이터
    config.py      환경변수 기반 설정
    extensions.py  DB, CORS, Migration, Swagger 확장 초기화
  migrations/      Alembic 마이그레이션
  tests/           pytest 기반 API 테스트
```

### 설계 포인트

- `create_app` 패턴을 사용해 앱 생성, 확장 초기화, blueprint 등록을 한 곳에서 관리했습니다.
- Route는 HTTP 입출력에 집중하고, 실제 도메인 처리는 Service 계층으로 위임했습니다.
- Repository 계층을 두어 SQLAlchemy 쿼리와 서비스 로직이 섞이지 않도록 했습니다.
- `ApiError` 기반 공통 예외 응답 구조를 만들어 프론트엔드가 `error.code` 중심으로 일관되게 처리할 수 있게 했습니다.
- JWT 인증, OAuth, 관리자 권한, 게시판/댓글, 파일 업로드, 이벤트 API를 모듈별 blueprint로 분리했습니다.
- Alembic 마이그레이션을 통해 DB 스키마 변경 이력을 코드로 관리합니다.
- 테스트 환경은 SQLite in-memory DB를 사용해 빠르게 API 단위 테스트를 수행할 수 있도록 구성했습니다.

### 주요 도메인

- Auth: 회원가입, 로그인, JWT 기반 인증, Google/Naver/Kakao OAuth
- Users/Admin: 사용자 정보 및 관리자 기능
- Posts/Comments: 공지, 문의, 버그 리포트 게시판과 댓글
- Files: Cloudflare R2 기반 업로드 및 presigned URL 처리
- CCTV: ITS CCTV API 연동
- Events: AI 탐지 이벤트 저장, 필터링, 상세 조회
- AI Lab: 프론트엔드 모델 데모와 Vision FastAPI 서버 사이의 중계 API

## 프론트엔드 구조

프론트엔드는 Next.js App Router와 TypeScript 기반으로 구성했습니다. 화면, 컴포넌트, API 클라이언트, 타입, 상태 관리를 분리해 기능이 늘어나도 탐색 가능한 구조를 유지했습니다.

```text
frontend/src/
  app/          App Router 페이지 및 API proxy route
  components/
    atoms/      Button, Input, Card, Badge 등 기본 UI
    molecules/  FormField, StatCard, SectionHeader 등 조합 컴포넌트
    organisms/  Header, Dashboard, Board, Admin, AI Lab 등 화면 단위 컴포넌트
    templates/  공통 레이아웃 템플릿
  services/     백엔드 API 호출 모듈
  stores/       Zustand 기반 전역 인증 상태
  hooks/        인증/관리자 접근 제어 훅
  types/        API 응답과 도메인 타입
  lib/          포맷팅, 유틸리티
```

### 설계 포인트

- 페이지 파일은 라우팅과 조립에 집중하고, 실제 화면 로직은 organism 컴포넌트로 분리했습니다.
- `services/*Api.ts`에서 API 호출을 캡슐화해 UI 컴포넌트가 fetch 세부 구현에 의존하지 않도록 했습니다.
- `types/`에 도메인 타입을 분리해 백엔드 응답 형태를 명시적으로 다룹니다.
- `useRequireAuth`, `useRequireAdmin` 훅으로 보호 페이지의 접근 제어를 재사용 가능하게 만들었습니다.
- Next.js API proxy route를 두어 프론트엔드에서 백엔드 API 경로를 안정적으로 중계할 수 있게 했습니다.
- Leaflet, HLS.js를 활용해 지도 기반 CCTV 탐색과 실시간 영상 재생 UX를 구현했습니다.
- AI Lab 화면은 샘플 이미지, 업로드 이미지, threshold 조절, 모델별 결과 표시를 분리된 컴포넌트로 구성했습니다.

## Worker 구조

Worker는 사용자 요청/응답 사이클과 분리된 백그라운드 처리 런타임입니다. CCTV 수집, 이미지 저장, 모델 추론, 이벤트 저장처럼 시간이 오래 걸리거나 실패 가능성이 있는 작업을 API 서버 밖으로 분리했습니다.

```text
worker/
  main.py           CCTV 수집 파이프라인 실행
  vision_client.py  Vision FastAPI 호출
  vlm_client.py     2차 판단 모델 연동 클라이언트
  event_writer.py   Backend Events API로 결과 저장/수정
```

Worker의 핵심 흐름은 다음과 같습니다.

1. ITS API에서 대상 CCTV 스트림 URL을 조회합니다.
2. ffmpeg를 우선 사용하고 실패 시 OpenCV fallback으로 프레임을 캡처합니다.
3. 캡처 이미지를 Cloudflare R2에 저장합니다.
4. Vision Model Server에 이미지를 보내 1차 탐지 결과를 받습니다.
5. 위험 후보 이벤트를 Backend API에 저장합니다.
6. 필요한 경우 VLM 2차 판단 결과를 이벤트에 반영합니다.

이 구조를 통해 API 서버는 사용자 요청 처리에 집중하고, 영상 수집/AI 추론 파이프라인은 독립적으로 운영할 수 있습니다.

## Vision Model Server

Vision Model Server는 FastAPI 기반의 별도 추론 서버입니다.

```text
vision-model/
  main.py       FastAPI 모델 서버
  predict.py    Cog 호환 Predictor
  cog.yaml      배포/실행 설정
  requirements.txt
```

모델은 RT-DETRv2 ONNX 형식으로 제공되며, 입력 이미지를 받아 객체 탐지 결과를 JSON으로 반환합니다. 탐지 클래스는 `fire`, `smoke`, `carlight`이며, 백엔드 AI Lab API와 Worker는 이 결과를 서비스 도메인에 맞게 변환해 사용합니다.

AI 추론 서버를 백엔드와 분리한 이유는 다음과 같습니다.

- GPU 의존성과 웹 API 의존성을 분리해 배포 환경을 유연하게 가져갈 수 있습니다.
- 모델 교체나 추론 최적화가 백엔드 도메인 API에 직접 영향을 주지 않습니다.
- 추론 서버 장애가 전체 사용자 API 장애로 바로 확산되는 것을 줄일 수 있습니다.

## 데이터 및 이벤트 설계

`Event` 모델은 CCTV 기반 위험 탐지 결과를 저장합니다.

주요 필드는 다음과 같습니다.

- `cctv_id`, `cctv_name`, `location_name`: 탐지 위치 식별 정보
- `detected_at`: 탐지 시각
- `risk_score`, `risk_candidate`: 1차 모델 기반 위험 점수와 후보 여부
- `is_fire`, `vlm_reason`: 2차 판단 결과와 사유
- `detected_classes`: 탐지된 클래스 목록
- `snapshot_key`: R2에 저장된 캡처 이미지 key

조회 성능을 고려해 `cctv_id + detected_at`, `detected_at`, `is_fire` 기준 인덱스를 두었습니다. 이벤트 목록 API는 페이지네이션, CCTV ID, 화재 여부, 기간 필터를 지원합니다.

## 테스트와 품질 관리

백엔드는 pytest 기반 테스트를 포함합니다.

- 사용자 API 테스트
- 게시글 API 테스트
- 댓글 API 테스트
- AI Lab과 Vision API 연동 테스트

특히 AI Lab 테스트에서는 실제 Vision 서버를 호출하지 않고 mock response를 주입해, 외부 서버 상태와 무관하게 백엔드 변환 로직을 검증합니다. 이는 실무에서 외부 API 의존성을 테스트 가능한 경계로 분리하는 방식과 같습니다.

## 실무 관점에서 보여주는 역량

이 프로젝트를 통해 다음 역량을 보여줄 수 있습니다.

- 단일 화면 구현을 넘어 프론트엔드, 백엔드, 워커, AI 서버를 연결한 end-to-end 서비스 설계
- Route, Service, Repository, Model 계층 분리를 통한 백엔드 유지보수성 확보
- 타입, API client, 상태 관리, 컴포넌트 계층화를 통한 프론트엔드 확장성 확보
- 외부 공공 API, Cloudflare R2, OAuth, JWT, Vision API 등 다양한 외부 시스템 연동 경험
- 실시간성/비동기성이 필요한 작업을 Worker로 분리하는 운영 관점의 설계
- DB 마이그레이션, 테스트, 공통 예외 응답, 환경변수 설정 등 기본적인 실무 개발 프로세스 이해
- AI 모델을 서비스 기능으로 통합하기 위한 API 경계 설계 및 결과 변환 경험

## 프로젝트에서 특히 신경 쓴 부분

- AI 모델을 단순 데모로 두지 않고 실제 서비스 이벤트 저장 흐름과 연결했습니다.
- 프론트엔드가 백엔드 구현 세부사항에 직접 묶이지 않도록 API client와 타입을 분리했습니다.
- 백엔드에서 도메인 로직과 DB 접근 로직을 분리해 기능 추가 시 수정 범위를 줄였습니다.
- CCTV 수집과 추론을 Worker로 분리해 사용자-facing API와 배치성 작업의 책임을 나눴습니다.
- 모델 서버를 독립 런타임으로 분리해 GPU 환경, 모델 교체, 추론 성능 개선을 별도로 다룰 수 있게 했습니다.

## 향후 개선 방향

- Worker 스케줄링을 Celery, RQ, APScheduler 등으로 고도화
- 이벤트 생성 API에 Worker 전용 인증 토큰 적용
- CCTV 수집 실패, Vision API 실패, R2 업로드 실패에 대한 재시도 정책 강화
- 운영 모니터링을 위한 structured logging 및 metrics 추가
- 이벤트 알림 기능 추가
- E2E 테스트와 프론트엔드 컴포넌트 테스트 보강
- 모델별 precision/recall, false positive 분석 대시보드 추가

## 한 줄 요약

Flare On The Road는 AI 모델, 백엔드 API, 백그라운드 워커, 프론트엔드 대시보드를 하나의 서비스로 연결한 프로젝트입니다. 단순 기능 구현보다 실무에서 중요한 관심사 분리, 외부 시스템 연동, 장애 범위 분리, 테스트 가능한 구조를 의식해 설계했습니다.
