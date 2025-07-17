# Aura AI: 자동 생성 기술 분석 보고서

## 1. 시스템 아키텍처

Aura AI는 **Flutter**로 구현된 프론트엔드, **FastAPI**로 구현된 백엔드, 그리고 **Supabase**를 데이터베이스 및 인증/BaaS 플랫폼으로 사용하는 현대적인 아키텍처로 구성되어 있습니다.

- **프론트엔드 (Flutter):** 사용자 인터페이스와 경험(UX)을 담당합니다. 클린 아키텍처와 BLoC 패턴을 적용하여 UI 로직과 비즈니스 로직을 분리했으며, `go_router`로 라우팅을, `get_it`으로 의존성 주입을 관리합니다.
- **백엔드 (FastAPI):** 프론트엔드와 외부 서비스(Google Gemini) 사이의 중재자 역할을 하는 퍼사드(Facade) 서버입니다. 복잡한 AI 관련 워크플로우를 오케스트레이션하고, 핵심 비즈니스 로직을 처리합니다.
- **데이터베이스 & BaaS (Supabase):** PostgreSQL 데이터베이스, 사용자 인증(Auth), 실시간 데이터 동기화 등 백엔드의 많은 기능을 위임받아 처리합니다. RLS(행 수준 보안)를 통해 데이터 접근을 안전하게 제어합니다.
- **LLM (Google Gemini):** 사용자의 자연어 입력을 이해하고 구조화된 계획을 생성하는 핵심 지능��� 제공합니다.

## 2. 디렉토리 구조

```
C:/Users/aslko/Desktop/programming/geminiCLI/mymanager/
├───backend/
│   ├───app/
│   │   ├───api/
│   │   │   └───v1/
│   │   │       ├───__init__.py
│   │   │       ├───planner_router.py
│   │   │       ├───profiles.py
│   │   │       ├───projects_router.py
│   │   │       └───tasks_router.py
│   │   ├───core/
│   │   ├───schemas/
│   │   └───services/
│   │   ├───__init__.py
│   │   └───main.py
│   ├───requirements.txt
│   └───config.py
└───frontend/
    ├───lib/
    │   ├───core/
    │   │   ├───di/
    │   │   └───routes/
    │   ├───data/
    │   │   ├───datasources/
    │   │   ├───models/
    │   │   └───repositories/
    │   ├───domain/
    │   │   ├───entities/
    │   │   └───repositories/
    │   ├───presentation/
    │   │   ├───bloc/
    │   │   └───pages/
    │   └───main.dart
    ├───pubspec.yaml
    └───analysis_options.yaml
```

## 3. 백엔드 분석 (FastAPI)

### 3.1. API 엔드포인트 명세

| 메서드 | 경로                      | 요청 모델      | 응답 모델      | 핵심 기능                               |
| :----- | :------------------------ | :------------- | :------------- | :-------------------------------------- |
| POST   | /api/v1/planner/summarize | `PlanInput`    | `ProjectCreate`| 자연어 입력을 분석하여 계획 초안 생성   |
| POST   | /api/v1/projects/         | `ProjectCreate`| `ProjectRead`  | 새로운 프로젝트 생성                    |
| POST   | /api/v1/tasks/            | `TaskCreate`   | `TaskRead`     | 새로운 태스크 생성                      |
| GET    | /api/v1/profiles/me       | -              | `Profile`      | 현재 사용자 프로필 조회                 |

*참고: 이 외에도 표준 CRUD 엔드포인트(GET, PATCH, DELETE)가 `projects`와 `tasks`에 존재할 것으로 예상됩니다.*

### 3.2. 핵심 서비스 로직 (PlannerService)

`planner_service.py`는 AI 기반 계획 생성의 전체 워크플로우를 담당하는 핵심 컴포넌트입니다.

1.  **퍼사드 패턴 적용:** `create_personalized_plan` 메소드는 AI 호출, DB 저장, 응답 생성을 하나의 단순한 인터페이스 뒤에 캡슐화합니다.
2.  **Gemini API 연동:** `GeminiClient`를 통해 사용자의 입력과 컨텍스트를 Gemini API에 전달하고, 구조화된 계획(제목, 요약, 태스크 목록)을 JSON 형태로 반환받습니다.
3.  **Supabase 연동:** `SupabaseClient`를 통해 AI가 생성한 계획 데이터를 PostgreSQL 데이터베이스에 저장합니다.

## 4. 프론트엔드 분석 (Flutter)

### 4.1. 아키텍처 패턴

- **클린 아키텍처:** 프로젝트는 명확하게 3개의 계층으로 분리되어 있습니다.
    - **Presentation:** BLoC 패턴을 사용하여 UI 상태를 관리하고, 사용자 이벤트를 처리합니다. (`PlannerBloc`, `HomePage`)
    - **Domain:** 비즈니스 로직의 핵심 규칙과 인터페이스를 정의합니다. (`PlannerRepository` 인터페이스)
    - **Data:** 외부(API, DB)와의 통신을 책임지고, Domain 계층의 인터페이스를 구현합니다. (`PlannerRepositoryImpl`, `PlannerRemoteDataSource`)
- **BLoC 패턴:** UI와 비즈니스 로직을 분리하기 위해 사용됩니다. UI는 `Event`를 `Bloc`에 보내고, `Bloc`은 로직 처리 후 `State`를 변경하여 UI를 업데이트합니다.

### 4.2. 데이터 흐름 (Planner 기능 예시)

`Planner` 기능에서 사용자가 자연어를 입력하고 AI의 요약 결과를 받기까지의 데이터 흐름은 다음과 같습니다.

1.  **View (`HomePage`):** 사용자가 텍스트를 입력하고 '계획 생성' 버튼을 누르면 `PlannerBloc.add(SummarizeInput(text))` 이벤트를 발생시킵니다.
2.  **Bloc (`PlannerBloc`):** `SummarizeInput` 이벤트를 수신하고, `Domain` 계층의 `PlannerRepository`에 작업를 위임합니다.
3.  **Repository (`PlannerRepositoryImpl`):** `Data` 계층의 `PlannerRemoteDataSource`를 호출하여 API 통신을 요청합니다.
4.  **Datasource (`PlannerRemoteDataSourceImpl`):** `Dio`를 사용하여 FastAPI 백엔드의 `/api/v1/planner/summarize` 엔드포인트를 호출하고, 응답 JSON을 데이터 모델(`ProjectCreate`)로 변환하여 반환합니다.
5.  **역방향 전파:** 변환된 데이터는 `Repository` -> `Bloc`을 거쳐 최종적으로 UI에 표시될 `PlannerState.summarized` 상태로 변경되어 `HomePage`에 전달됩니다.
