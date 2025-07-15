# Windsurf Todo Demo - 코드 품질 규칙

> Kent Beck의 BPlusTree3 코드 품질 표준을 React + TypeScript 환경에 맞게 적용

## 🚫 절대 금지 사항 (NEVER)

### 1. 런타임 에러 발생 코드
```typescript
// ❌ NEVER - 런타임 에러 발생 가능
const todo = todos[index]; // index가 범위를 벗어날 수 있음
const id = parseInt(input); // NaN 반환 가능
throw new Error("This should never happen"); // 프로덕션에서 panic

// ✅ DO - 안전한 접근
const todo = todos.at(index) ?? null;
const id = Number.isNaN(parseInt(input)) ? null : parseInt(input);
const result = safeTodoOperation(todo);
```

### 2. any 타입 남용
```typescript
// ❌ NEVER - 타입 안전성 포기
const handleData = (data: any) => { /* ... */ };
const response = await fetch(url) as any;

// ✅ DO - 명시적 타입 정의
interface TodoData {
  id: string;
  title: string;
  completed: boolean;
}
const handleData = (data: TodoData) => { /* ... */ };
const response: ApiResponse<TodoData> = await fetch(url).then(r => r.json());
```

### 3. 에러 무시 또는 숨김
```typescript
// ❌ NEVER - 에러 무시
try {
  await saveTodo(todo);
} catch {
  // 에러 무시
}

// ❌ NEVER - 에러 숨김
const result = riskyOperation().catch(() => null);

// ✅ DO - 명시적 에러 처리
try {
  await saveTodo(todo);
} catch (error) {
  console.error('Todo 저장 실패:', error);
  showErrorMessage('할일 저장에 실패했습니다.');
  return { success: false, error };
}
```

### 4. 상태 불일치 가능성
```typescript
// ❌ NEVER - 상태 직접 변경
state.todos.push(newTodo); // 불변성 위반
user.profile.name = newName; // 직접 변경

// ✅ DO - 불변성 유지
setState(prev => ({ ...prev, todos: [...prev.todos, newTodo] }));
setUser(prev => ({ ...prev, profile: { ...prev.profile, name: newName } }));
```

---

## ✅ 필수 준수 사항 (ALWAYS)

### 1. 테스트 우선 개발 (TDD)
```typescript
// ✅ 실패하는 테스트 먼저 작성
describe('TodoStore.addTodo', () => {
  it('should add todo with valid title', () => {
    const store = new TodoStore();
    const result = store.addTodo('New Todo');
    
    expect(result.success).toBe(true);
    expect(store.getTodos()).toHaveLength(1);
    expect(store.getTodos()[0].title).toBe('New Todo');
  });

  it('should reject empty title', () => {
    const store = new TodoStore();
    const result = store.addTodo('');
    
    expect(result.success).toBe(false);
    expect(result.error?.type).toBe('INVALID_TITLE');
  });
});
```

### 2. 불변성 검증 포함
```typescript
// ✅ 데이터 구조 불변성 보장
class TodoStore {
  private todos: readonly Todo[] = [];

  addTodo(title: string): Result<Todo, TodoError> {
    if (!this.isValidTitle(title)) {
      return { success: false, error: new TodoError('INVALID_TITLE', 'Title cannot be empty') };
    }

    const newTodo = this.createTodo(title);
    this.todos = [...this.todos, newTodo]; // 불변성 유지
    
    // 불변성 검증
    this.validateInvariants();
    
    return { success: true, data: newTodo };
  }

  private validateInvariants(): void {
    // ID 중복 검사
    const ids = this.todos.map(t => t.id);
    if (new Set(ids).size !== ids.length) {
      throw new Error('Invariant violation: Duplicate todo IDs');
    }
  }
}
```

### 3. 타입 가드 및 유효성 검사
```typescript
// ✅ 런타임 타입 검증
function isTodo(value: unknown): value is Todo {
  return (
    typeof value === 'object' &&
    value !== null &&
    typeof (value as any).id === 'string' &&
    typeof (value as any).title === 'string' &&
    typeof (value as any).completed === 'boolean'
  );
}

function validateTodoInput(input: unknown): Result<TodoInput, ValidationError> {
  if (!input || typeof input !== 'object') {
    return { success: false, error: new ValidationError('Input must be an object') };
  }

  const { title } = input as any;
  if (typeof title !== 'string' || title.trim().length === 0) {
    return { success: false, error: new ValidationError('Title must be a non-empty string') };
  }

  return { success: true, data: { title: title.trim() } };
}
```

### 4. 관심사 분리 (Clean Architecture)
```typescript
// ✅ 계층별 책임 분리

// Domain Layer - 비즈니스 로직
interface Todo {
  id: string;
  title: string;
  completed: boolean;
  createdAt: Date;
  updatedAt: Date;
}

class TodoEntity {
  constructor(private todo: Todo) {}

  toggle(): TodoEntity {
    return new TodoEntity({
      ...this.todo,
      completed: !this.todo.completed,
      updatedAt: new Date()
    });
  }

  updateTitle(title: string): Result<TodoEntity, TodoError> {
    if (!title.trim()) {
      return { success: false, error: new TodoError('INVALID_TITLE', 'Title cannot be empty') };
    }

    return {
      success: true,
      data: new TodoEntity({
        ...this.todo,
        title: title.trim(),
        updatedAt: new Date()
      })
    };
  }
}

// Application Layer - 유스케이스
class TodoService {
  constructor(private repository: TodoRepository) {}

  async addTodo(title: string): Promise<Result<Todo, TodoError>> {
    const validation = this.validateTitle(title);
    if (!validation.success) {
      return validation;
    }

    const todo = this.createTodo(title);
    const saveResult = await this.repository.save(todo);
    
    if (!saveResult.success) {
      return { success: false, error: new TodoError('SAVE_FAILED', 'Failed to save todo') };
    }

    return { success: true, data: todo };
  }
}

// Infrastructure Layer - 데이터 저장
class LocalStorageTodoRepository implements TodoRepository {
  async save(todo: Todo): Promise<Result<void, StorageError>> {
    try {
      const todos = await this.getAll();
      const updated = [...todos.filter(t => t.id !== todo.id), todo];
      localStorage.setItem('todos', JSON.stringify(updated));
      return { success: true, data: undefined };
    } catch (error) {
      return { success: false, error: new StorageError('SAVE_FAILED', error) };
    }
  }
}

// Presentation Layer - UI 컴포넌트
function TodoComponent({ todoService }: { todoService: TodoService }) {
  const [title, setTitle] = useState('');
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(null);

    const result = await todoService.addTodo(title);
    if (result.success) {
      setTitle('');
    } else {
      setError(result.error.message);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="할일을 입력하세요"
      />
      <button type="submit">추가</button>
      {error && <div className="error">{error}</div>}
    </form>
  );
}
```

---

## 🧪 테스트 요구사항

### 포괄적 테스트 커버리지 (≥90%)
```typescript
// ✅ 단위 테스트
describe('TodoEntity', () => {
  it('should toggle completion status', () => {
    const todo = new TodoEntity(createMockTodo({ completed: false }));
    const toggled = todo.toggle();
    
    expect(toggled.isCompleted()).toBe(true);
    expect(toggled.getUpdatedAt()).toBeInstanceOf(Date);
  });
});

// ✅ 통합 테스트
describe('TodoService Integration', () => {
  it('should save todo to repository', async () => {
    const mockRepo = new MockTodoRepository();
    const service = new TodoService(mockRepo);
    
    const result = await service.addTodo('Test Todo');
    
    expect(result.success).toBe(true);
    expect(mockRepo.getSavedTodos()).toHaveLength(1);
  });
});

// ✅ 속성 기반 테스트 (Property-based Testing)
import fc from 'fast-check';

describe('TodoStore Properties', () => {
  it('should maintain invariants under any sequence of operations', () => {
    fc.assert(
      fc.property(
        fc.array(fc.oneof(
          fc.record({ type: fc.constant('add'), title: fc.string() }),
          fc.record({ type: fc.constant('toggle'), id: fc.string() }),
          fc.record({ type: fc.constant('delete'), id: fc.string() })
        )),
        (operations) => {
          const store = new TodoStore();
          
          operations.forEach(op => {
            switch (op.type) {
              case 'add':
                if (op.title.trim()) store.addTodo(op.title);
                break;
              case 'toggle':
                store.toggleTodo(op.id);
                break;
              case 'delete':
                store.deleteTodo(op.id);
                break;
            }
          });

          // 불변성 검증
          expect(store.isValid()).toBe(true);
          expect(store.getTodos().every(t => t.id && t.title)).toBe(true);
        }
      )
    );
  });
});
```

### 메모리 및 성능 테스트
```typescript
// ✅ 메모리 누수 테스트
describe('Memory Management', () => {
  it('should not leak memory with large datasets', () => {
    const store = new TodoStore();
    const initialMemory = process.memoryUsage().heapUsed;

    // 대량 데이터 처리
    for (let i = 0; i < 10000; i++) {
      store.addTodo(`Todo ${i}`);
    }

    // 데이터 정리
    store.clear();
    
    // 가비지 컬렉션 강제 실행
    if (global.gc) global.gc();
    
    const finalMemory = process.memoryUsage().heapUsed;
    const memoryIncrease = finalMemory - initialMemory;
    
    // 메모리 증가가 합리적 범위 내인지 확인
    expect(memoryIncrease).toBeLessThan(1024 * 1024); // 1MB 미만
  });
});

// ✅ 성능 벤치마크
describe('Performance', () => {
  it('should handle 1000 todos within acceptable time', () => {
    const store = new TodoStore();
    const startTime = performance.now();

    for (let i = 0; i < 1000; i++) {
      store.addTodo(`Todo ${i}`);
    }

    const endTime = performance.now();
    const duration = endTime - startTime;

    expect(duration).toBeLessThan(100); // 100ms 미만
  });
});
```

---

## 🏗 아키텍처 요구사항

### Result 타입 패턴
```typescript
// ✅ 명시적 에러 처리를 위한 Result 타입
type Result<T, E = Error> = 
  | { success: true; data: T }
  | { success: false; error: E };

// 사용 예시
async function fetchTodos(): Promise<Result<Todo[], ApiError>> {
  try {
    const response = await fetch('/api/todos');
    if (!response.ok) {
      return { 
        success: false, 
        error: new ApiError('FETCH_FAILED', `HTTP ${response.status}`) 
      };
    }
    
    const data = await response.json();
    return { success: true, data };
  } catch (error) {
    return { 
      success: false, 
      error: new ApiError('NETWORK_ERROR', error) 
    };
  }
}
```

### 에러 타입 정의
```typescript
// ✅ 구체적인 에러 타입
export enum TodoErrorType {
  INVALID_TITLE = 'INVALID_TITLE',
  NOT_FOUND = 'NOT_FOUND',
  STORAGE_ERROR = 'STORAGE_ERROR',
  VALIDATION_ERROR = 'VALIDATION_ERROR'
}

export class TodoError extends Error {
  constructor(
    public readonly type: TodoErrorType,
    message: string,
    public readonly cause?: unknown
  ) {
    super(message);
    this.name = 'TodoError';
  }

  static invalidTitle(title: string): TodoError {
    return new TodoError(
      TodoErrorType.INVALID_TITLE,
      `Invalid title: "${title}". Title must be a non-empty string.`
    );
  }

  static notFound(id: string): TodoError {
    return new TodoError(
      TodoErrorType.NOT_FOUND,
      `Todo with id "${id}" not found.`
    );
  }
}
```

---

## ✅ 코드 완성 체크리스트

코드를 완성으로 표시하기 전 다음을 확인:

### 컴파일 및 정적 분석
- [ ] TypeScript 컴파일 오류 없음
- [ ] ESLint 경고 없음
- [ ] Prettier 포맷팅 적용
- [ ] 모든 import/export 정상 동작

### 테스트 및 품질
- [ ] 모든 테스트 통과 (단위, 통합, E2E)
- [ ] 테스트 커버리지 ≥90%
- [ ] 속성 기반 테스트 포함
- [ ] 메모리 누수 테스트 통과

### 아키텍처 및 설계
- [ ] 관심사 분리 원칙 준수
- [ ] SOLID 원칙 적용
- [ ] 의존성 주입 패턴 사용
- [ ] 에러 처리 일관성 유지

### 성능 및 안정성
- [ ] 메모리 사용량 예측 가능
- [ ] 모든 코드 경로에서 데이터 무결성 보장
- [ ] 성능 벤치마크 허용 범위 내
- [ ] 불변성 검증 통과

### 문서화
- [ ] 모든 공개 API 문서화
- [ ] 복잡한 로직 주석 추가
- [ ] 에러 케이스 문서화
- [ ] 사용 예시 포함

---

## 📚 문서화 표준

### API 문서화
```typescript
/**
 * Todo 항목을 안전하게 추가합니다.
 * 
 * @param title - Todo 제목 (빈 문자열 불가)
 * @returns 성공 시 생성된 Todo, 실패 시 에러 정보
 * 
 * @example
 * ```typescript
 * const result = await todoService.addTodo("새로운 할일");
 * if (result.success) {
 *   console.log('Todo 생성됨:', result.data.id);
 * } else {
 *   console.error('생성 실패:', result.error.message);
 * }
 * ```
 * 
 * @throws Never throws - all errors returned via Result type
 */
async addTodo(title: string): Promise<Result<Todo, TodoError>> {
  // 구현
}
```

### 에러 문서화
```typescript
/**
 * Todo 관련 에러를 나타내는 클래스
 * 
 * @example
 * ```typescript
 * try {
 *   const result = await todoService.addTodo("");
 *   if (!result.success) {
 *     switch (result.error.type) {
 *       case TodoErrorType.INVALID_TITLE:
 *         showValidationError(result.error.message);
 *         break;
 *       case TodoErrorType.STORAGE_ERROR:
 *         showStorageError(result.error.message);
 *         break;
 *     }
 *   }
 * }
 * ```
 */
export class TodoError extends Error {
  // 구현
}
```

---

## 🎯 30분 라이브 코딩 적용 가이드

### Phase 1 (5분) - 프로젝트 설정
- [ ] TypeScript strict 모드 활성화
- [ ] ESLint 규칙 설정
- [ ] 테스트 프레임워크 설정
- [ ] 기본 타입 정의

### Phase 2 (20분) - 핵심 구현
- [ ] Result 타입 패턴 적용
- [ ] 테스트 우선 개발
- [ ] 에러 처리 일관성 유지
- [ ] 불변성 보장

### Phase 3 (5분) - 품질 검증
- [ ] 테스트 실행 및 커버리지 확인
- [ ] 정적 분석 도구 실행
- [ ] 성능 간단 검증
- [ ] 문서화 완성

---

*이 규칙들은 Kent Beck의 품질 철학을 React + TypeScript 환경에 맞게 적용한 것으로, 30분 라이브 코딩에서도 실용적으로 사용할 수 있도록 최적화되었습니다.*
