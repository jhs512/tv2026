# MVP 에디터 설계 — 2026-03-15

## 목적

마크다운 텍스트를 입력하면 `!scene` 구분자로 씬을 파싱하고, 실시간으로 미리보기하는 최소 기능 웹 애플리케이션.

## 범위

- 파서: `!scene`으로 씬 분리 (최소 DSL)
- 렌더러: `text` 씬 타입만 (검정 배경, 흰 텍스트, 중앙 정렬, 16:9 비율)
- 에디터: `<textarea>` (라이브러리 없음)
- 레이아웃: 좌우 분할 (에디터 50% / 미리보기 50%)
- 상태 관리: Zustand

## 아키텍처

### 디렉토리 구조

```
front/src/
├── app/
│   └── page.tsx                  # 좌우 분할 레이아웃 진입점
├── features/
│   └── editor/
│       ├── components/
│       │   ├── EditorPane.tsx    # textarea 에디터
│       │   └── PreviewPane.tsx   # 씬 목록 렌더링
│       └── store.ts              # Zustand 스토어
├── domain/
│   ├── parser/
│   │   ├── parseScenes.ts        # 순수 파서 함수
│   │   └── parseScenes.test.ts   # Guard
│   └── scene/
│       └── types.ts              # Scene 타입 정의
└── components/
    └── scene/
        └── TextScene.tsx         # text 씬 렌더러
```

### 데이터 흐름

```
textarea 입력 → parseScenes() → Scene[] → Zustand store → PreviewPane 렌더링
```

## 도메인 타입

```typescript
// domain/scene/types.ts
type SceneType = 'text'

interface Scene {
  id: string      // 'scene-0', 'scene-1', ...
  type: SceneType
  content: string // !scene 이후 ~ 다음 !scene 전까지의 텍스트 (trim)
}
```

## 파서 규칙

- `!scene` 라인을 만나면 새 씬 시작
- `!scene` 라인 자체는 content에 포함하지 않음
- `!scene` 없이 텍스트만 있으면 빈 배열 반환
- 빈 입력이면 빈 배열 반환

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
  setMarkdown: (md: string) => void  // 호출 시 parseScenes() 자동 실행
}
```

## UI 컴포넌트

### page.tsx

좌우 50/50 분할 레이아웃. `EditorPane`과 `PreviewPane`을 나란히 배치.

### EditorPane

- `<textarea>` full height
- `onChange` → `store.setMarkdown()` 호출

### PreviewPane

- `scenes` 배열 순회하며 `TextScene` 렌더링
- 씬이 없으면 안내 문구 표시

### TextScene

- 16:9 고정 비율
- 검정 배경(`#000`), 흰 텍스트(`#fff`), 중앙 정렬
- content를 `<p>`로 표시

## Rules (Guard 대상)

1. `parseScenes`는 순수 함수여야 한다 — 부수 효과 없음
2. 도메인 타입(`Scene`)은 `domain/` 아래에만 정의한다
3. 컴포넌트는 파싱 로직을 직접 포함하지 않는다

## Guard (테스트 케이스)

```typescript
// parseScenes.test.ts
it('!scene 구분자로 씬을 분리한다')
it('!scene 라인은 content에 포함하지 않는다')
it('빈 입력이면 빈 배열을 반환한다')
it('!scene 없이 텍스트만 있으면 빈 배열을 반환한다')
it('content는 trim 처리된다')
```

## 제외 범위 (향후 이터레이션)

- 씬 타입 확장 (code, image, video 등)
- 스타일 속성 (background, color, align)
- 애니메이션 / 전환 효과
- 익스포트 (MP4, GIF)
- 에디터 라이브러리 (CodeMirror 등)
- `!var`, `!chapter` 등 부가 디렉티브
