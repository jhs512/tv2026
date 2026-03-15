# MVP 에디터 설계 — 2026-03-15

## 목적

마크다운 텍스트를 입력하면 `!scene` 구분자로 씬을 파싱하고, 실시간으로 미리보기하는 최소 기능 웹 애플리케이션.

## 범위

- 파서: `!scene`으로 씬 분리 (최소 DSL)
- 렌더러: `text` 씬 타입만 (검정 배경, 흰 텍스트, 중앙 정렬, 16:9 비율)
- 에디터: `<textarea>` (라이브러리 없음)
- 레이아웃: 좌우 분할 (에디터 50% / 미리보기 50%)
- 상태 관리: Zustand
- 테스트 러너: Vitest (dev dependency로 설치, `vitest.config.ts` 구성 필요)

### 선행 조건

Guard 테스트 실행 전 아래 작업이 완료되어야 한다:
1. `vitest`, `@vitejs/plugin-react` dev dependency 설치
2. `front/vitest.config.ts` 생성 (최소 내용: `plugins: [react()]`, `test.environment: 'node'`)
3. `package.json`에 `"test": "vitest"` 스크립트 추가

## 아키텍처

### 디렉토리 구조

```
front/src/
├── app/
│   └── page.tsx                       # 좌우 분할 레이아웃 진입점
├── features/
│   └── editor/
│       ├── components/
│       │   ├── EditorPane.tsx         # textarea 에디터
│       │   ├── PreviewPane.tsx        # 씬 목록 렌더링
│       │   └── TextScene.tsx          # text 씬 렌더러
│       └── store.ts                   # Zustand 스토어
└── domain/
    ├── parser/
    │   ├── parseScenes.ts             # 순수 파서 함수
    │   └── parseScenes.test.ts        # Guard
    └── scene/
        └── types.ts                   # Scene 타입 정의
```

> `TextScene`은 에디터 기능에 종속된 컴포넌트이므로 `features/editor/components/` 아래에 배치한다. 향후 씬 타입이 추가되더라도 씬 렌더러는 동일한 위치에 배치한다.

### 데이터 흐름

```
textarea 입력 → parseScenes() → Scene[] → Zustand store → PreviewPane 렌더링
```

## 도메인 타입

```typescript
// domain/scene/types.ts
type SceneType = 'text'

interface Scene {
  id: string      // 'scene-0', 'scene-1', ... (parseScenes가 index 기반으로 생성)
  type: SceneType
  content: string // !scene 이후 ~ 다음 !scene 전까지의 텍스트 (trim, 내부 줄바꿈 보존)
}
```

## 파서 규칙

- `!scene` 라인을 만나면 새 씬 시작
- `!scene` 라인 자체는 content에 포함하지 않음
- `!scene` 없이 텍스트만 있으면 빈 배열 반환
- 빈 입력이면 빈 배열 반환
- content는 앞뒤 공백을 trim 처리 (내부 줄바꿈은 보존)
- trim 후 content가 빈 문자열인 씬도 유효한 씬으로 포함한다
- `id`는 `parseScenes` 함수가 배열 인덱스 기반으로 할당한다 (`'scene-0'`, `'scene-1'`, ...)

### DSL 예시

```
!scene
안녕하세요

!scene
두 번째 씬입니다
```

→ `Scene[]` 2개 생성

## 상태 관리 (Zustand)

```typescript
interface EditorStore {
  markdown: string
  scenes: Scene[]
  setMarkdown: (md: string) => void
}

// 구현:
setMarkdown: (md) => set({ markdown: md, scenes: parseScenes(md) })
```

`setMarkdown` 호출 시 `parseScenes(md)`를 즉시 실행하여 `scenes`를 동기적으로 갱신한다.

## UI 컴포넌트

### page.tsx

좌우 50/50 분할 레이아웃. `EditorPane`과 `PreviewPane`을 나란히 배치.

### EditorPane

- `<textarea>` full height
- `onChange` → `store.setMarkdown()` 호출

### PreviewPane

- `scenes` 배열 순회하며 `TextScene` 렌더링
- 씬이 없으면 안내 문구 표시
- 세로 스크롤 가능 (`overflow-y: auto`), 각 씬은 세로로 쌓임

### TextScene

- 16:9 고정 비율
- 검정 배경(`#000`), 흰 텍스트(`#fff`), 중앙 정렬
- content를 `<p>`로 표시 (내부 줄바꿈은 `white-space: pre-wrap` 적용)

## Rules

1. `parseScenes`는 순수 함수여야 한다 — 부수 효과 없음
2. 도메인 타입(`Scene`)은 `domain/` 아래에만 정의한다
3. 컴포넌트는 파싱 로직을 직접 포함하지 않는다

## Guard (테스트 케이스)

Guard 테스트는 path alias 없이 상대 경로로 import한다 (tsconfig의 `paths` 미사용). Vitest root는 `front/` 디렉토리 기준이다.

Rule 2, 3은 파일 내용을 `fs.readFileSync`로 읽어 import 경로를 정규식으로 검증하는 방식으로 구현한다.

```typescript
// domain/parser/parseScenes.test.ts (Vitest)

// Rule 1: 파서 동작
it('!scene 구분자로 씬을 분리한다')
it('!scene 라인은 content에 포함하지 않는다')
it('빈 입력이면 빈 배열을 반환한다')
it('!scene 없이 텍스트만 있으면 빈 배열을 반환한다')
it('content는 trim 처리된다')
it('id는 scene-0, scene-1 순서로 할당된다')
it('공백만 있는 씬 body도 씬으로 포함된다')
it('내부 빈 줄은 content에 보존된다')

// Rule 2: 타입 경계 — EditorPane, PreviewPane, TextScene, store.ts는
//          Scene 타입을 domain/scene/types에서만 import한다
it('EditorPane은 Scene 타입을 domain/scene/types에서만 import한다')
it('PreviewPane은 Scene 타입을 domain/scene/types에서만 import한다')
it('TextScene은 Scene 타입을 domain/scene/types에서만 import한다')
it('store.ts는 Scene 타입을 domain/scene/types에서만 import한다')

// Rule 3: 컴포넌트 파싱 금지
it('EditorPane은 parseScenes를 직접 import하지 않는다')
it('PreviewPane은 parseScenes를 직접 import하지 않는다')
it('TextScene은 parseScenes를 직접 import하지 않는다')
```

## 제외 범위 (향후 이터레이션)

- 씬 타입 확장 (code, image, video 등)
- 스타일 속성 (background, color, align)
- 애니메이션 / 전환 효과
- 익스포트 (MP4, GIF)
- 에디터 라이브러리 (CodeMirror 등)
- `!var`, `!chapter` 등 부가 디렉티브
