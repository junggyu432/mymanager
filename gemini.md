너는 20년차 베타랑 프로젝트 매니저야. overview.md 파일과 schema.md파일과 ing.md파일을 읽고 난뒤 다음 명령을 따라줘
수정할 때에는 지시사항만 따르지말고 수정시 문제가 있는지 더 좋은 방법이 있는 지 고민하고 지시사항과 다르게 할 거라면 나에게 물어봐줘

Flutter 중복 API 호출(409 Conflict) 문제 해결 가이드
문제 현상: [Project로 만들기] 버튼 클릭 시, POST /projects API가 두 번 호출되어, 첫 번째는 성공(201)하고 두 번째는 중복 데이터로 인해 실패(409 Conflict) 에러가 발생함.
근본 원인: UI의 상태 관리 미흡으로 인해, 비동기 작업(API 호출)이 진행 중임에도 불구하고 사용자가 버튼을 다시 클릭할 수 있거나, 다른 요인에 의해 이벤트가 중복으로 발생하고 있음.
목표: API 요청이 단 한 번만 실행되도록 보장하고, 처리 중에는 사용자에게 명확한 시각적 피드백을 제공하여 중복 입력을 방지한다.
AI 어시스턴트 작업 지시서
핵심 임무: PlannerBloc의 상태를 활용하여, API 호출이 진행 중일 때는 모든 관련 UI 액션을 비활성화(Disable)하는 '상태 잠금(State Locking)' 메커니즘을 구현한다.
단계 1: PlannerBloc 상태 확인 및 보장
파일 열기: frontend/lib/presentation/bloc/planner_bloc/planner_bloc.dart
CreateProject 이벤트 핸들러 확인:
on<CreateProject> 핸들러의 가장 첫 줄에 emit(const PlannerState.loading()); 코드가 있는지 반드시 확인한다.
이 코드가 있어야, 버튼을 누르는 즉시 앱의 상태가 '로딩 중'으로 변경되어 UI가 반응할 수 있다.
Generated dart
// on<CreateProject> 핸들러 내부
on<CreateProject>((event, emit) async {
  // ★★★ 1. 즉시 로딩 상태를 emit하여 UI를 잠근다 ★★★
  emit(const PlannerState.loading()); 
  
  final result = await _createProjectUseCase(
      projectData: event.projectData, token: event.token);
  
  result.fold(
    (failure) => emit(PlannerState.error(message: failure.message)),
    // ★★★ 2. 작업 완료 후, 최종 상태를 emit하여 UI를 해제한다 ★★★
    (createdProject) => emit(PlannerState.projectCreated(project: createdProject)),
  );
});
Use code with caution.
Dart
단계 2: HomePage UI 로직 수정 (가장 중요)
임무: BLoC의 loading 상태를 감지하여, 모든 활성 버튼을 비활성화하고 시각적 피드백을 제공한다.
파일 열기: frontend/lib/presentation/pages/home_page.dart (또는 home_view.dart)
BlocBuilder 또는 BlocConsumer 내부 수정:
builder 콜백 함수의 가장 위에서, 현재 상태가 로딩 중인지 판단하는 bool 변수를 만든다.
Generated dart
builder: (context, state) {
  // ★★★ 현재 상태가 로딩 중인지 명확히 판단 ★★★
  final bool isLoading = state.maybeWhen(
    loading: () => true,
    orElse: () => false,
  );
  // ...
}
Use code with caution.
Dart
모든 버튼의 onPressed 콜백 수정:
[Project로 만들기], [Task로 만들기], 그리고 [Analyze Input] 버튼까지, API를 호출하는 모든 버튼의 onPressed 속성을 아래와 같이 수정한다.
isLoading이 true일 때, onPressed에 null을 할당하여 버튼을 비활성화시킨다.
Generated dart
// 예시: [Project로 만들기] 버튼
ElevatedButton(
  // ★★★ 로딩 중일 때는 null을 할당하여 버튼 비활성화 ★★★
  onPressed: isLoading ? null : () {
    // ... 기존의 createProject 이벤트 발생 로직 ...
  },
  child: const Text('Create as Project'),
)
Use code with caution.
Dart
(선택적, 그러나 강력 추천) 로딩 인디케이터 추가:
사용자에게 "지금 처리 중"임을 명확히 알려주기 위해, isLoading일 때 버튼의 child를 CircularProgressIndicator로 변경한다.
Generated dart
// 예시: [Project로 만들기] 버튼의 child
child: isLoading 
    ? const SizedBox(
        height: 20, width: 20, 
        child: CircularProgressIndicator(strokeWidth: 2, color: Colors.white),
      ) 
    : const Text('Create as Project'),
Use code with caution.
Dart
단계 3: BlocListener 로직 확인
BlocConsumer의 listener 부분을 확인한다.
projectCreated 또는 taskCreated 상태가 감지되었을 때, UI 상태를 초기화하는 로직(예: _textController.clear(), Bloc 상태를 initial로 되돌리는 이벤트 호출 등)이 포함되어 있는지 확인한다. 이 부분이 있어야, 첫 번째 작업이 성공적으로 끝난 후 사용자가 바로 다음 작업을 이어서 할 수 있다.