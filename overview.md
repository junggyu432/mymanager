Aura AI: 프로젝트 최종 청사진 및 개발 로드맵
1. 서비스 핵심 비전
"너만을 위한 매니저, Aura AI"
서비스 정의: LLM 기반의 지능형 대화형 일정 관리 및 생활 계획 서비스. 사용자의 막연한 생각을 구체적이고 실행 가능한 계획으로 전환하고, 그 실행 과정을 함께하는 동기부여 파트너.
2. 최종 아키텍처 청사진 (Architecture Blueprint)
시스템 구성도 (System Diagram)
Generated code
[Flutter Frontend] <=> [FastAPI Backend] <=> [Google Gemini API]
      ^                     ^
      | (Auth, Realtime)    | (DB CRUD, Auth)
      v                     v
    [Supabase] -----------+
Use code with caution.
기술 스택 (Tech Stack)
Frontend (빠릿한 비서): Flutter
아키텍처: 클린 아키텍처 (Presentation/Domain/Data) + MVI (BLoC)
상태 관리: flutter_bloc
라우팅: go_router
네트워킹: dio
DI (의존성 주입): get_it
Backend (공감하는 두뇌): FastAPI (Python)
역할: 복잡한 비즈니스 로직, AI 워크플로우 오케스트레이션(Facade), LLM 연동
서버: uvicorn
Database & BaaS (데이터 창고/인증 센터): Supabase
역할: 데이터베이스(PostgreSQL), 사용자 인증, 실시간 동기화, 기본 CRUD API
LLM (지능의 원천): Google Gemini API
모델: 작업의 복잡도에 따라 Flash (빠른 응답) / Pro (고품질 생성) 모델 선택적 사용
핵심 디자인 패턴
클린 아키텍처: 프론트엔드와 백엔드 모두 계층 분리를 통해 유지보수성 및 확장성 확보.
리포지토리 패턴: 데이터 소스에 대한 접근을 추상화하여 비즈니스 로직과 데이터 로직 분리.
퍼사드 패턴: 복잡한 AI 워크플로우(LLM 호출, DB 저장, 데이터 가공)를 백엔드 Service 계층에서 단순한 인터페이스로 캡슐화.
어댑터 패턴: 데이터 계층의 Model(API 응답)을 도메인 계층의 Entity(비즈니스 객체)로 변환.
3. 개발 계획 개요 (Development Roadmap)
우리는 '수직 분할(Vertical Slice)' 전략에 따라, 전체 시스템을 관통하는 작은 기능을 먼저 완성하여 아키텍처의 유효성을 검증하고 점진적으로 확장해 나간다.
Phase 0: 기초 공사 및 아키텍처 리팩토링 (현재 단계)
목표: 새로운 청사진에 맞춰 프로젝트의 뼈대를 세우고, 핵심 라이브러리를 설치한다.
주요 작업:
Git 브랜치 생성 (feature/refactor-clean-architecture).
Backend: Flask -> FastAPI 마이그레이션 및 폴더 구조 리팩토링.
Frontend: 클린 아키텍처 기반 폴더 구조 생성 및 BLoC, GetIt 등 핵심 패키지 설정.
.env 파일 설정 및 config.py 연동.
Supabase 프로젝트 생성 및 테이블 스키마(users, projects, tasks) 초기 설계.
Phase 1: 사용자 인증 흐름 완성 (첫 번째 수직 분할)
목표: 사용자가 앱을 사용하기 위한 가장 기본적인 기능인 '회원가입' 및 '로그인'을 End-to-End로 완성한다.
주요 작업:
Frontend: 회원가입/로그인 UI 페이지 및 BLoC 로직 구현. (변경 없음)
Backend/Supabase: Supabase의 인증 기능을 활용하여 사용자 가입 및 로그인 처리. (변경 없음)
Backend (FastAPI): 클라이언트로부터 Authorization: Bearer <JWT> 헤더를 받아서 서비스 계층으로 전달하는 구조만 설정합니다. (핵심 변경)
로컬 토큰 검증 로직은 구현하지 않습니다. FastAPI는 토큰을 투명하게 전달하는 역할만 합니다.
End-to-End 테스트: Flutter 앱에서 회원가입 → Supabase DB에 유저 생성 → 로그인 → JWT 토큰 획득 → 획득한 JWT로 보호된 FastAPI 테스트 엔드포인트(예: /users/me) 호출 성공까지의 전체 흐름을 검증합니다.
Phase 2: 핵심 기능 구현 - "AI 기반 계획 생성"
목표: Aura AI의 핵심 가치인 '대화형 계획 생성' 기능을 구현한다.
주요 작업:
Frontend: 사용자가 아이디어를 입력하는 대화형 UI 및 생성된 계획/태스크를 보여주는 화면 구현.
Backend (Facade 패턴 적용):
PlannerService에서 Gemini API와 Supabase Client를 오케스트레이션하는 로직 구현.
사용자 입력 -> Gemini 아이디어 생성 -> 구조화된 데이터(Plan/Tasks) 파싱 -> Supabase DB 저장의 워크플로우를 완성.
End-to-End 테스트: Flutter 앱에서 "주말 데이트 뭐하지?" 입력 -> FastAPI 백엔드 호출 -> Gemini 응답 -> Supabase에 계획 저장 -> 앱 화면에 결과 표시까지의 흐름을 검증한다.
Phase 3: 계획 관리 및 실행
목표: 생성된 계획을 사용자가 관리하고 실행할 수 있도록 지원한다.
주요 작업:
Frontend/Supabase: Supabase의 실시간 기능을 활용하여 태스크 완료 상태(체크박스)를 동기화.
Frontend: 생성된 계획 목록 조회, 상세 보기, 태스크 수정/삭제 UI 및 로직 구현.
Backend: 계획 수정/삭제 등 관리 API 엔드포인트 구현.
Phase 4: 고도화 및 감성적 터치 추가
목표: '매니저'로서의 역할을 강화하고 앱의 완성도를 높인다.
주요 작업:
Backend/LLM: 사용자의 컨텍스트(과거 계획, 성향)를 활용하여 더 개인화된 응답을 생성하는 로직 고도화.
Backend/Frontend: 지능형 알람 및 동기부여 메시지 생성 및 푸시 알림 기능 구현.
UI/UX: 앱 전반의 애니메이션, 전환 효과 등 감성적인 디자인 요소 추가.



Aura AI: 프로젝트 마스터 플랜 (The Grand Blueprint)
1. 프로젝트 비전 및 목적 (The Vision & Purpose)
1.1. 비전 (Vision)
"단순한 '앱'을 넘어, 사용자의 곁에서 함께 성장하는 '지능형 파트너'를 제공한다."
1.2. 미션 (Mission)
우리는 사용자의 머릿속에 떠다니는 막연한 생각과 복잡한 목표를, 명확하고 실행 가능한 계획으로 전환하고, 그 실행 과정을 함께하며 동기를 부여함으로써 사용자의 잠재력 실현을 돕는다.
1.3. 핵심 가치 (Core Values)
공감 (Empathy): 사용자의 언어와 숨은 의도를 이해한다.
지능 (Intelligence): 복잡한 문제를 구체적인 단계로 분해하고 조언한다.
동행 (Companionship): 계획의 전 과정에서 사용자를 지지하고 격려한다.

2. 시스템 아키텍처 및 기술 스택 (The Architecture & Tech Stack)
**"최고의 전문가에게 맡기고, 우리는 핵심에 집중한다"**는 철학 아래, 각 컴포넌트의 역할과 책임을 명확히 분리한 마이크로서비스 지향 아키텍처를 채택한다.

2.1. 전체 시스템 구성도
Generated code
[ User (Flutter App) ]
     │
     ├─(인증)───> [ 🔐 Supabase Auth (JWT 발급/관리) ]
     │
     └─(API 요청 w/ JWT)───> [ ♣️ API Gateway (Routing) ]
          │
          ├─> [ 🧠 Core API (FastAPI) ] ───(PostgREST)──> [ ⚡ Supabase ]
          │      (CRUD, 상태 관리)                          (DB, RLS)
          │
          └─> [ 🤖 LLM Service (FastAPI) ] ───(API)───> [ ✨ Google Gemini ]
                 (AI 로직, 대화 관리)        │
                                           └─(Task Queue)──> [ 📩 Celery & Redis ]

2.2. 기술 스택 (The Toolbox)
Frontend (The Face):
프레임워크: Flutter (Dart) - 단일 코드로 iOS/Android 동시 지원
상태 관리: flutter_bloc - 예측 가능한 상태 관리
라우팅: go_router - 선언적 라우팅
네트워킹: dio - 강력한 HTTP 클라이언트
의존성 주입: get_it - 서비스 로케이터 패턴
Backend (The Brains):
프레임워크: FastAPI (Python) - 현대적이고 빠른 비동기 웹 프레임워크
서버: Uvicorn - 고성능 ASGI 서버
데이터 검증: Pydantic - 타입 기반 데이터 검증 및 설정 관리
비동기 작업: Celery & Redis - 푸시 알림, 리밸런싱 등 백그라운드 작업 처리
인프라 및 플랫폼 (The Foundation):
BaaS (Backend-as-a-Service): Supabase
데이터베이스: PostgreSQL - 안정적이고 강력한 관계형 데이터베이스
인증: Supabase Auth - JWT 기반의 안전한 사용자 인증 시스템
데이터 API: PostgREST - RLS 정책이 적용된 자동 생성 REST API
AI 엔진: Google Gemini API
모델: gemini-1.5-flash-latest (또는 최신 버전) - 속도와 성능의 균형
핵심 기술: JSON Mode를 활용한 구조화된 데이터 생성
개발 및 배포 (DevOps):
버전 관리: Git & GitHub
컨테이너화: Docker - 일관된 개발 및 배포 환경
CI/CD: GitHub Actions - 자동화된 테스트 및 배포 파이프라인
모니터링: Grafana & Loki (향후 도입 고려) - 시스템 상태 모니터링 및 로깅

3. 서비스 및 컴포넌트 역할 정의 (The Roles & Responsibilities)
Flutter App: 사용자와의 모든 상호작용을 책임지는 인터페이스. Supabase Auth를 통해 인증을 직접 처리하고, JWT를 관리하며, API Gateway를 통해 백엔드 서비스에 비즈니스 로직을 요청한다.
API Gateway: 모든 외부 요청이 거쳐가는 교통 관제소. 요청 경로에 따라 적절한 내부 서비스(Core API, LLM Service)로 투명하게 전달(Proxy Pass)하는 역할을 한다. (로컬에서는 Mocking, 프로덕션에서는 NGINX, Kong 등 사용)
Core API (FastAPI): 시스템의 중앙 데이터 관리자. 사용자의 JWT를 받아 PostgREST API를 통해 데이터베이스의 CRUD(생성, 읽기, 수정, 삭제)를 수행한다. 데이터의 일관성과 상태 변경 로직을 책임진다. DB에 직접 연결하지 않는다.
LLM Service (FastAPI): 프로젝트의 AI 두뇌. 사용자의 자연어 입력을 받아 Gemini API를 호출하고, 그 결과를 분석/요약/생성하는 모든 지능적인 작업을 전담한다.
Supabase Platform: 우리 서비스의 핵심 인프라 파트너.
Auth: '여권 발급처'이자 '입국 심사관'. 사용자 인증과 JWT 발급을 책임진다.
Database (PostgreSQL): 모든 데이터를 저장하는 '안전 금고'.
PostgREST & RLS: '보안 요원' 겸 '은행 창구'. RLS 규칙에 따라 JWT 소유자의 데이터만 안전하게 입출금하는 유일한 공식 창구.
Celery & Redis: 보이지 않는 곳에서 궂은일을 하는 지원 부대. 푸시 알림 전송, 주기적인 Task 리밸런싱 등 시간이 걸리거나 예약된 작업을 백그라운드에서 묵묵히 처리한다.

Aura AI: 기능 및 정책 명세서 (Features & Policies)
문서 버전: 1.0
최종 수정일: 2025-07-16
관련 문서: 프로젝트 마스터 플랜 v5.0
1. 서비스 기능 및 사용자 경험(UX) 명세
핵심 원칙: "사용자가 생각하게 두지 말고, 자연스럽게 이끈다."
1.1. 핵심 기능: "생각의 씨앗을 계획의 나무로 키우는 과정"
입력 단계 (Seeding):
UI: 앱 메인 화면은 영감을 주는 배경과 함께 "무엇을 계획해 볼까요?" 라는 하나의 텍스트 필드만 존재. 키보드가 올라오면 "예시: 3개월 안에 바디 프로필 찍기" 같은 동기부여 문구가 Placeholder로 나타남.
사용자 행동: 자유로운 자연어 입력 (AI 계획) 또는 하단의 + 버튼을 통한 수동 입력 (빠른 등록).
분석 및 제안 단계 (Germination):
AI 프로세스: 사용자가 [AI로 계획 세우기]를 선택하면, 백엔드는 POST /planner/summarize를 호출.
AI의 결과물: SummarizedContent (제목, 설명, 날짜, 반복 규칙, 추가 질문 등) JSON 객체.
UI: 사용자에게 AI의 분석 결과를 명확한 카드 형태로 제시.
"AI가 당신의 생각을 이렇게 정리했어요. 어떠신가요?"
제목: [AI가 추출한 제목]
설명: [AI가 추출한 설명]
시간 정보: [AI가 추출한 시간 정보 또는 "언제로 설정할까요?"]
사용자 선택: 사용자는 이 카드의 내용을 수정할 수 있으며, 최종적으로 [이대로 프로젝트 만들기] 또는 [이대로 할 일 만들기] 버튼을 선택.
구체화 단계 (Branching):
트리거: 사용자가 저장된 Project/Task 상세 화면에서 [🤖 AI로 세분화하기] 버튼을 클릭.
AI 프로세스: 백엔드는 POST /tasks/{id}/decompose를 호출. 이때 **사용자 프로필(직업, 나이 등)**을 함께 전달.
AI의 결과물: 하위 Task 목록 (JSON Array). 각 Task는 content와 tips를 포함.
개인화 예시: 직업: 학생 -> "관련 논문 리서치", 직업: 개발자 -> "기술 스택 리서치"
UI: AI가 제안한 하위 Task 목록을 '편집 가능한 체크리스트' 형태로 보여줌. 사용자는 각 항목을 수정/삭제/추가하고 순서를 변경할 수 있음.
사용자 행동: [이대로 모두 추가] 버튼을 눌러 최종 저장.
1.2. 사용자 여정 지도 (User Journey Map)
1단계 (인식 및 설치): "내 삶을 체계적으로 관리하고 싶다" -> 스토어 검색 -> 앱 설치.
2단계 (온보딩): 회원가입 -> "Aura AI가 당신을 더 잘 이해할 수 있도록 알려주세요" 라는 화면에서 선택적인 프로필 정보 입력 (직업, 관심사 등). -> 핵심 기능 튜토리얼 (1~2 화면).
3단계 (첫 가치 경험): "영어 공부 다시 시작하기" 입력 -> AI가 "매일 단어 10개 외우기", "주 1회 쉐도잉" 등 구체적인 계획을 제안 -> 사용자는 "와, 막막했는데 할 수 있겠다"는 느낌을 받음.
4.단계 (일상적 사용): 매일 앱을 열어 오늘의 할 일 확인 -> 완료 체크 -> AI의 동기부여 메시지 수신 -> 새로운 생각 입력.
5단계 (충성도 형성): 누적된 완료 기록과 AI의 긍정적 피드백을 보며 성취감을 느낌 -> 유료 구독(Pro) 전환 고려.
2. LLM 서비스 상세 로직
핵심 원칙: "AI는 유능한 비서이지, 주인이 아니다."
2.1. 프롬프트 엔지니어링 전략: "페르소나 + 컨텍스트 + 지시"
모든 프롬프트는 아래 3가지 요소를 조합하여 구성한다.
페르소나(Persona): "당신은 세계 최고의 지능형 계획 비서 'Aura AI'입니다. 항상 긍정적이고, 명확하며, 사용자를 지지하는 톤앤매너를 유지하세요."
컨텍스트(Context):
사용자 프로필: [사용자 프로필: { "직업": "디자이너", "관심사": ["여행", "요리"] }]
이전 대화 요약: (필요 시) [이전 대화 요약: 사용자는 '포트폴리오 제작' 프로젝트에 대해 이야기하고 있었습니다.]
명확한 지시(Instruction): "이 Task를 완수하기 위한 구체적인 하위 단계 3가지를 제안하세요. 디자이너라는 직업을 고려하여, '레퍼런스 수집'과 같은 단계를 포함하세요."
2.2. 대화 맥락(Context) 관리 전략
단기 기억 (Session-based): 하나의 계획을 생성하는 대화(턴) 동안에는, 프론트엔드(Bloc)가 대화 기록과 중간 결과(요약된 내용 등)를 상태로 관리하고, 매번 API 요청 시 서버에 전달한다. (Stateless Backend)
장기 기억 (Long-term Memory):
1단계 (DB 저장): 모든 확정된 Project/Task는 PostgreSQL에 저장한다.
2단계 (RAG 도입 검토 - Phase 5 이후): 사용자가 "예전에 내가 했던 여행 계획 스타일로 짜줘" 같은 요청을 할 때, PostgreSQL에 저장된 과거 데이터를 검색하여 관련 내용을 프롬프트에 포함시키는 RAG(Retrieval-Augmented Generation) 패턴 도입을 장기적으로 검토한다. 이를 위해 pgvector 같은 PostgreSQL 확장 기능 활용을 고려한다.
2.3. AI 안전장치 (Safety Guardrails)
프롬프트 레벨: 모든 프롬프트 끝에 **"절대 생성해서는 안 되는 내용"**에 대한 명시적인 지시를 추가한다. "재정적, 의학적, 법률적 조언을 하지 마세요. 비윤리적이거나 위험한 활동을 제안하지 마세요. 사용자를 비난하거나 부정적인 감정을 유발하는 표현을 사용하지 마세요."
아웃풋 필터링: Gemini API의 내장된 safety_settings을 BLOCK_MEDIUM_AND_ABOVE 수준으로 설정한다.
키워드 기반 차단: 응답에 특정 금지 키워드(예: 극단적 선택, 불법 등)가 포함될 경우, 해당 응답을 사용자에게 보여주지 않고 "죄송해요, 그 주제에 대해서는 조언해 드릴 수 없어요." 와 같은 기본 메시지로 대체하는 로직을 백엔드에 추가한다.
3. 보안 및 안정성 강화 계획
핵심 원칙: "사전에 방지하고, 지속적으로 감시하며, 신속하게 대응한다."
3.1. API 어뷰징 방지
API Gateway (NGINX/Kong):
Rate Limiting: IP 주소 기준, 사용자 ID 기준으로 분당/시간당 API 호출 횟수를 제한한다. (예: 분당 60회)
LLM API 엔드포인트 특별 관리: 비용과 직접적으로 연결되는 /planner/... 엔드포인트는 더 엄격한 Rate Limiting 정책을 적용한다. (예: 분당 5회)
3.2. 데이터 보안
전송 계층: 모든 통신은 HTTPS (TLS 1.2 이상)를 강제한다.
저장 계층 (Encryption at Rest):
1단계 (Supabase 기본): Supabase는 기본적으로 디스크 레벨 암호화를 제공한다.
2단계 (필드 레벨 암호화 - Phase 4 이후): 사용자의 개인적인 description이나 content 필드는, 백엔드 서버에서 암호화 라이브러리(cryptography)를 사용하여 암호화한 뒤 DB에 저장하고, 조회 시 복호화하는 방안을 고려한다. 이를 위해 .env에 별도의 ENCRYPTION_KEY 관리가 필요하다.
API 키: 사용자가 개인 API 키를 입력하는 기능 구현 시, DB 저장 전 반드시 양방향 암호화하여 저장한다.
3.3. 장애 복구 및 모니터링
모니터링 대상 (Key Metrics):
API Gateway/Backend: API 호출당 평균 지연 시간(Latency), 에러율(HTTP 5xx, 4xx), 초당 요청 수(RPS).
LLM Service: Gemini API 호출 평균 지연 시간, 실패율, 토큰 사용량.
Database: CPU/Memory 사용률, 활성 연결 수.
Alerting Rule:
"5분간 5xx 에러율이 1%를 초과하면, PagerDuty/Slack으로 즉시 경고 알림을 보낸다."
"Gemini API 호출 실패율이 5%를 초과하면, 경고 알림을 보낸다."
로깅 (Logging): 모든 API 요청과 중요한 비즈니스 로직 실행 시, 사용자 ID를 포함한 구조화된 로그(JSON 형식)를 stdout으로 출력한다. Loki는 이 로그들을 수집하여 검색 및 분석이 가능하게 한다.