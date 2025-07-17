Aura AI: 데이터 및 API 명세서 v1.1 (최종 확정판)
문서 버전: 1.1
최종 수정일: 2025-07-16
관련 문서: 프로젝트 마스터 플랜 v5.0, 기능 및 정책 명세서 v1.0
핵심 변경 사항: 반복 규칙 구조화, Sub-task/Checklist 역할 정의, API 페이지네이션, 상세 오류 처리 도입
1. 데이터 스키마 명세 (The Data Blueprint - SQL)
원칙: 데이터는 정규화되고, 관계는 명확하며, 성능을 위해 인덱싱한다. 비정형 데이터를 배제하고, 모든 것은 구조화한다.
1.1. SQL 실행 스크립트 (전체)
(이 스크립트는 Supabase SQL Editor에 바로 실행할 수 있는 최종본입니다)
Generated sql
-- Aura AI 데이터베이스 스키마 v1.1 (FINAL)
-- 실행 전 주의: 이 스크립트는 기존 관련 테이블을 모두 삭제하고 새로 생성합니다.

-- ==== 1. 초기화: 기존 객체 삭제 ====
DROP TABLE IF EXISTS public.task_tags, public.tags, public.checklists, public.tasks, public.projects, public.profiles CASCADE;
DROP TYPE IF EXISTS public.project_status;

-- ==== 2. ENUM 타입 생성 ====
CREATE TYPE public.project_status AS ENUM ('in_progress', 'completed', 'archived');
COMMENT ON TYPE public.project_status IS 'Defines the status of a project.';

-- ==== 3. 테이블 생성 ====

CREATE TABLE public.profiles (
  id uuid NOT NULL PRIMARY KEY,
  updated_at timestamptz DEFAULT timezone('utc'::text, now()),
  username text UNIQUE,
  full_name text,
  avatar_url text,
  job text, -- 개인화를 위한 사용자 직업
  preferences jsonb,
  CONSTRAINT username_length CHECK (char_length(username) >= 3)
);
COMMENT ON TABLE public.profiles IS 'User profile information, linked to auth.users.';

CREATE TABLE public.projects (
  id bigserial PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  title text NOT NULL,
  description text,
  status public.project_status DEFAULT 'in_progress'::public.project_status NOT NULL,
  created_at timestamptz DEFAULT timezone('utc'::text, now()) NOT NULL
);
COMMENT ON TABLE public.projects IS 'A container or folder for grouping top-level tasks.';

CREATE TABLE public.tasks (
  id bigserial PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  project_id bigint REFERENCES public.projects(id) ON DELETE CASCADE,
  parent_task_id bigint REFERENCES public.tasks(id) ON DELETE CASCADE,
  content text NOT NULL,
  is_completed boolean DEFAULT false NOT NULL,
  scheduled_at timestamptz,
  repeat_rule jsonb, -- [수정] text에서 jsonb로 변경하여 구조화
  tips jsonb,
  created_at timestamptz DEFAULT timezone('utc'::text, now()) NOT NULL
);
COMMENT ON COLUMN public.tasks.repeat_rule IS 'Structured recurrence rule. e.g., {"type": "weekly", "days": [1, 3, 5]}';
COMMENT ON TABLE public.tasks IS 'Hierarchical to-do items. Sub-tasks are themselves Tasks.';

CREATE TABLE public.checklists (
  id bigserial PRIMARY KEY,
  task_id bigint NOT NULL REFERENCES public.tasks(id) ON DELETE CASCADE,
  content text NOT NULL,
  is_checked boolean DEFAULT false NOT NULL,
  created_at timestamptz DEFAULT timezone('utc'::text, now()) NOT NULL
);
COMMENT ON TABLE public.checklists IS 'Simple, non-schedulable sub-items for completing a single task.';

CREATE TABLE public.tags (
  id bigserial PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  name text NOT NULL,
  color text,
  UNIQUE(user_id, name)
);

CREATE TABLE public.task_tags (
  task_id bigint NOT NULL REFERENCES public.tasks(id) ON DELETE CASCADE,
  tag_id bigint NOT NULL REFERENCES public.tags(id) ON DELETE CASCADE,
  PRIMARY KEY (task_id, tag_id)
);

-- ==== 4. 자동화 및 보안 (RLS, 트리거) ====

-- 4.1. 자동 프로필 생성을 위한 함수 및 트리거

-- 함수: handle_new_user
-- 역할: Supabase Auth에 새로운 사용자가 가입(INSERT)하면, 이 함수가 자동으로 호출되어
--       public.profiles 테이블에 해당 사용자의 프로필 레코드를 생성합니다.
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  -- 새 사용자의 id를 profiles 테이블의 id로 하여 새 행을 삽입합니다.
  INSERT INTO public.profiles (id)
  VALUES (new.id);
  RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
COMMENT ON FUNCTION public.handle_new_user() IS 'When a new user signs up, this function creates a corresponding profile record.';

-- 트리거: on_auth_user_created
-- 역할: auth.users 테이블에 INSERT 이벤트가 발생한 후, 각 행에 대해 handle_new_user 함수를 실행합니다.
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE PROCEDURE public.handle_new_user();


-- 4.2. 행 수준 보안 (Row Level Security - RLS) 정책

-- 먼저 모든 테이블에 RLS를 활성화합니다.
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.checklists ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.tags ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.task_tags ENABLE ROW LEVEL SECURITY;

-- 정책: Profiles
-- 설명: 사용자는 자신의 프로필 정보만 보거나 수정할 수 있습니다.
CREATE POLICY "Users can view and manage their own profile."
ON public.profiles FOR ALL
TO authenticated
USING (auth.uid() = id)
WITH CHECK (auth.uid() = id);

-- 정책: Projects
-- 설명: 사용자는 자신이 생성한 프로젝트에 대해서만 모든 작업을 수행할 수 있습니다.
CREATE POLICY "Users can manage their own projects."
ON public.projects FOR ALL
TO authenticated
USING (auth.uid() = user_id)
WITH CHECK (auth.uid() = user_id);

-- 정책: Tasks
-- 설명: 사용자는 자신이 생성한 할 일에 대해서만 모든 작업을 수행할 수 있습니다.
CREATE POLICY "Users can manage their own tasks."
ON public.tasks FOR ALL
TO authenticated
USING (auth.uid() = user_id)
WITH CHECK (auth.uid() = user_id);

-- 정책: Checklists
-- 설명: 사용자는 자신의 Task에 속한 체크리스트에 대해서만 모든 작업을 수행할 수 있습니다.
--       (Task 테이블의 user_id를 참조하여 정책을 설정합니다.)
CREATE POLICY "Users can manage checklists for their own tasks."
ON public.checklists FOR ALL
TO authenticated
USING ( (SELECT user_id FROM public.tasks WHERE id = checklists.task_id) = auth.uid() );

-- 정책: Tags
-- 설명: 사용자는 자신이 생성한 태그에 대해서만 모든 작업을 수행할 수 있습니다.
CREATE POLICY "Users can manage their own tags."
ON public.tags FOR ALL
TO authenticated
USING (auth.uid() = user_id)
WITH CHECK (auth.uid() = user_id);

-- 정책: Task_Tags (태그 연결)
-- 설명: 사용자는 자신의 Task에 대해서만 태그를 연결하거나 해제할 수 있습니다.
CREATE POLICY "Users can manage tag links for their own tasks."
ON public.task_tags FOR ALL
TO authenticated
USING ( (SELECT user_id FROM public.tasks WHERE id = task_tags.task_id) = auth.uid() );

-- ==== 5. 인덱스 생성 (성능 최적화) ====
CREATE INDEX idx_tasks_user_id_is_completed ON public.tasks(user_id, is_completed);
CREATE INDEX idx_tasks_user_id_scheduled_at ON public.tasks(user_id, scheduled_at);
CREATE INDEX idx_tasks_project_id ON public.tasks(project_id);
CREATE INDEX idx_tasks_parent_task_id ON public.tasks(parent_task_id);

SELECT 'Aura AI Database Schema v1.1 setup complete.';
Use code with caution.
SQL
1.2. Sub-tasks vs. Checklists 역할 정의
Checklists: 실행의 보조 도구. Task를 완료하기 위한 단순 단계 목록. (예: "장보기" Task의 Checklist -> "우유", "계란")
Sub-tasks: 독립적인 할 일. 부모 Task를 구성하는 더 작은 작업. 모든 Task와 동일한 속성(시간, 반복, 하위 Task 등)을 가질 수 있음. (예: "보고서 작성" Task의 Sub-task -> "자료 조사")
AI의 기본 생성물: AI의 "세분화" 기능은 기본적으로 Sub-tasks를 생성합니다.
2. API 명세 (The API Contract - v1.1)
원칙: API는 리소스 기반으로 설계하고, 페이로드 부풀림을 방지하며, 명확한 오류 코드를 반환한다. 모든 엔드포인트는 인증(JWT)을 필수로 요구한다.
2.1. 공통 사항
Base URL: /api/v1
인증: 모든 요청의 Authorization 헤더에 Bearer <JWT> 포함 필수.
표준 오류 응답 형식: {"error_code": "ERROR_CODE_NAME", "detail": "User-friendly message"}
2.2. AI 플래너 (/planner)
POST /planner/summarize
설명: 자연어 입력을 분석하여 구조화된 데이터(제목, 설명, 시간 정보 등)로 변환.
요청 본문: {"text": "string"}
응답 (200 OK): SummarizedContent 스키마.
주요 오류: LLM_PARSING_FAILED(503), INPUT_TOO_LONG(413)
POST /tasks/{task_id}/decompose
설명: 특정 Task를 하위 Task들로 분해하도록 AI에 요청.
요청 본문 (선택적): {"context_prompt": "string"} (예: "기획자의 관점에서")
응답 (200 OK): {"sub_tasks": [ { "content": "...", "tips": "..." }, ... ]}
2.3. 프로젝트 (/projects)
POST /
설명: 새 프로젝트 생성.
요청 본문: ProjectCreate 스키마.
응답 (201 Created): ProjectRead 스키마.
GET /
설명: 사용자의 모든 프로젝트 목록 조회 (페이지네이션).
쿼리 파라미터: ?page=1&size=20
응답 (200 OK): {"total": int, "page": int, "size": int, "items": [ProjectRead]}
GET /{project_id}
설명: 특정 프로젝트의 상세 정보 조회. (Task 목록 미포함)
응답 (200 OK): ProjectRead 스키마.
GET /{project_id}/tasks
설명: 특정 프로젝트에 속한 Task 목록 조회 (페이지네이션).
쿼리 파라미터: ?page=1&size=20
응답 (200 OK): {"total": int, "page": int, "size": int, "items": [TaskRead]}
PATCH /{project_id}
설명: 프로젝트 정보(제목, 설명, 상태) 수정.
요청 본문: ProjectUpdate 스키마.
응답 (200 OK): 수정된 ProjectRead 스키마.
DELETE /{project_id}
설명: 프로젝트 삭제. (장기적으로 비동기 처리 고려)
응답 (204 No Content): 성공적으로 삭제됨.
2.4. 태스크 (/tasks)
POST /
설명: 새 Task 생성. project_id 또는 parent_task_id를 포함할 수 있음.
요청 본문: TaskCreate 스키마.
응답 (201 Created): TaskRead 스키마.
GET /
설명: 사용자의 Task를 필터링하여 조회 (페이지네이션).
쿼리 파라미터: ?is_completed=false, ?date=YYYY-MM-DD, ?project_id=int, ?tag_id=int, ?page=1&size=20
응답 (200 OK): {"total": int, ..., "items": [TaskRead]}
(기타): GET /{task_id}, PATCH /{task_id}, DELETE /{task_id} 등 표준 CRUD 엔드포인트 포함.
(Checklists, Tags에 대한 CRUD API도 위와 유사한 패턴으로 설계)