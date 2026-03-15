# MVP 에디터 Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 마크다운 텍스트를 `!scene`으로 파싱해 좌우 분할 실시간 미리보기를 제공하는 MVP 웹 앱 구현.

**Architecture:** 순수 파서 함수(`parseScenes`) → Zustand 스토어 → React 컴포넌트로 이어지는 단방향 흐름. 도메인 타입은 `domain/` 아래에만 정의하고 컴포넌트는 파싱 로직을 가지지 않는다.

**Tech Stack:** Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS v4, Zustand, Vitest

---

## File Map

| 파일 | 역할 | 작업 |
|------|------|------|
| `front/package.json` | 의존성 | 수정 (Zustand, Vitest 추가) |
| `front/vitest.config.ts` | Vitest 설정 | 생성 |
| `front/src/domain/scene/types.ts` | Scene 타입 정의 | 생성 |
| `front/src/domain/parser/parseScenes.ts` | 순수 파서 함수 | 생성 |
| `front/src/domain/parser/parseScenes.test.ts` | Guard 테스트 | 생성 |
| `front/src/features/editor/store.ts` | Zustand 스토어 | 생성 |
| `front/src/features/editor/components/TextScene.tsx` | text 씬 렌더러 | 생성 |
| `front/src/features/editor/components/EditorPane.tsx` | textarea 에디터 | 생성 |
| `front/src/features/editor/components/PreviewPane.tsx` | 씬 목록 렌더링 | 생성 |
| `front/src/app/page.tsx` | 좌우 분할 레이아웃 | 수정 |

---

## Chunk 1: 환경 설정

### Task 1: 패키지 설치 및 Vitest 설정

**Files:**
- Modify: `front/package.json`
- Create: `front/vitest.config.ts`

- [ ] **Step 1: Zustand, Vitest, @vitejs/plugin-react 설치**

```bash
cd front
pnpm add zustand
pnpm add -D vitest @vitejs/plugin-react
```

Expected: `package.json`의 `dependencies`에 `zustand`, `devDependencies`에 `vitest`, `@vitejs/plugin-react` 추가됨.

- [ ] **Step 2: vitest.config.ts 생성**

`front/vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import { fileURLToPath } from 'url'
import path from 'path'

const __dirname = fileURLToPath(new URL('.', import.meta.url))

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
  test: {
    environment: 'node',
  },
})
```

> `@vitejs/plugin-react`는 React Compiler 없이 기본 Babel transform만 사용한다. Next.js의 `babel-plugin-react-compiler`는 Next.js 빌드에서만 적용되며 Vitest와 무관하다.

- [ ] **Step 3: package.json에 test 스크립트 추가**

`front/package.json`의 `scripts`에 추가:
```json
"test": "vitest"
```

- [ ] **Step 4: 설치 확인**

```bash
cd front
pnpm exec vitest run
```

Expected: "No test files found" 또는 0 tests (에러 없이 종료).

- [ ] **Step 5: 커밋 & 푸시**

```bash
git add front/package.json front/pnpm-lock.yaml front/vitest.config.ts
git commit -m "chore: vitest, zustand 설치 및 테스트 환경 설정"
git push
```

---

## Chunk 2: 도메인 레이어

### Task 2: Scene 타입 정의

**Files:**
- Create: `front/src/domain/scene/types.ts`

- [ ] **Step 1: 타입 파일 생성**

`front/src/domain/scene/types.ts`:
```typescript
export type SceneType = 'text'

export interface Scene {
  id: string
  type: SceneType
  content: string
}
```

- [ ] **Step 2: 커밋 & 푸시**

```bash
git add front/src/domain/scene/types.ts
git commit -m "feat: Scene 도메인 타입 정의"
git push
```

---

### Task 3: Guard 테스트 작성 (파서 + 아키텍처 Rule 2, 3 스텁)

**Files:**
- Create: `front/src/domain/parser/parseScenes.test.ts`

Rule 2, 3 Guard 스텁을 구현 전에 먼저 작성한다 (RGB 원칙: Guard → Build).

- [ ] **Step 1: Guard 테스트 파일 생성**

`front/src/domain/parser/parseScenes.test.ts`:
```typescript
import { readFileSync } from 'fs'
import { fileURLToPath } from 'url'
import { resolve } from 'path'
import { describe, it, expect } from 'vitest'
import { parseScenes } from './parseScenes'

const __dirname = fileURLToPath(new URL('.', import.meta.url))
const SRC = resolve(__dirname, '../..')

function readSrc(relativePath: string): string {
  return readFileSync(resolve(SRC, relativePath), 'utf-8')
}

// ─────────────────────────────────────────────
// Rule 1: 파서 동작
// ─────────────────────────────────────────────
describe('parseScenes — Rule 1: 파서 동작', () => {
  it('!scene 구분자로 씬을 분리한다', () => {
    const result = parseScenes('!scene\n첫 번째\n\n!scene\n두 번째')
    expect(result).toHaveLength(2)
  })

  it('!scene 라인은 content에 포함하지 않는다', () => {
    const result = parseScenes('!scene\n안녕하세요')
    expect(result[0].content).toBe('안녕하세요')
  })

  it('빈 입력이면 빈 배열을 반환한다', () => {
    expect(parseScenes('')).toEqual([])
  })

  it('!scene 없이 텍스트만 있으면 빈 배열을 반환한다', () => {
    expect(parseScenes('그냥 텍스트')).toEqual([])
  })

  it('content는 trim 처리된다', () => {
    const result = parseScenes('!scene\n  앞뒤 공백  \n')
    expect(result[0].content).toBe('앞뒤 공백')
  })

  it('id는 scene-0, scene-1 순서로 할당된다', () => {
    const result = parseScenes('!scene\nA\n\n!scene\nB')
    expect(result[0].id).toBe('scene-0')
    expect(result[1].id).toBe('scene-1')
  })

  it('공백만 있는 씬 body도 씬으로 포함된다', () => {
    const result = parseScenes('!scene\n   \n\n!scene\n내용')
    expect(result).toHaveLength(2)
    expect(result[0].content).toBe('')
  })

  it('내부 빈 줄은 content에 보존된다', () => {
    const result = parseScenes('!scene\n줄1\n\n줄2')
    expect(result[0].content).toBe('줄1\n\n줄2')
  })
})

// ─────────────────────────────────────────────
// Rule 2: Scene 타입은 domain/scene/types에서만 import
// ─────────────────────────────────────────────
describe('Rule 2: Scene 타입은 domain/scene/types에서만 import', () => {
  const files: [string, string][] = [
    ['features/editor/components/EditorPane.tsx', 'EditorPane'],
    ['features/editor/components/PreviewPane.tsx', 'PreviewPane'],
    ['features/editor/components/TextScene.tsx', 'TextScene'],
    ['features/editor/store.ts', 'store'],
  ]

  for (const [filePath, name] of files) {
    it(`${name}은 Scene 타입을 domain/scene/types에서만 import한다`, () => {
      const content = readSrc(filePath)
      // import { ... Scene ... } from '경로' 패턴 검사
      const importRegex = /import\s+[^;]*\bScene\b[^;]*from\s+['"]([^'"]+)['"]/g
      let match: RegExpExecArray | null
      while ((match = importRegex.exec(content)) !== null) {
        expect(match[1], `${name}의 Scene import 경로가 잘못됨`).toContain('domain/scene/types')
      }
    })
  }
})

// ─────────────────────────────────────────────
// Rule 3: 컴포넌트는 parseScenes를 직접 import하지 않는다
// ─────────────────────────────────────────────
describe('Rule 3: 컴포넌트는 parseScenes를 직접 import하지 않는다', () => {
  const files: [string, string][] = [
    ['features/editor/components/EditorPane.tsx', 'EditorPane'],
    ['features/editor/components/PreviewPane.tsx', 'PreviewPane'],
    ['features/editor/components/TextScene.tsx', 'TextScene'],
  ]

  for (const [filePath, name] of files) {
    it(`${name}은 parseScenes를 직접 import하지 않는다`, () => {
      const content = readSrc(filePath)
      expect(content, `${name}이 parseScenes를 직접 import하고 있음`).not.toMatch(
        /from\s+['"][^'"]*parseScenes['"]/
      )
    })
  }
})
```

- [ ] **Step 2: 테스트 실행 — Rule 1 실패, Rule 2/3 실패 확인**

```bash
cd front
pnpm exec vitest run src/domain/parser/parseScenes.test.ts
```

Expected: FAIL — `Cannot find module './parseScenes'` (파서 미구현)

- [ ] **Step 3: 커밋 & 푸시**

```bash
git add front/src/domain/parser/parseScenes.test.ts
git commit -m "guard: parseScenes Rule 1/2/3 Guard 테스트 추가"
git push
```

---

### Task 4: parseScenes 구현

**Files:**
- Create: `front/src/domain/parser/parseScenes.ts`

- [ ] **Step 1: 파서 구현**

`front/src/domain/parser/parseScenes.ts`:
```typescript
import { Scene } from '../scene/types'

export function parseScenes(markdown: string): Scene[] {
  if (!markdown.trim()) return []

  const lines = markdown.split('\n')
  const scenes: Scene[] = []
  let currentLines: string[] = []
  let inScene = false

  const flushScene = () => {
    if (inScene) {
      scenes.push({
        id: `scene-${scenes.length}`,
        type: 'text',
        content: currentLines.join('\n').trim(),
      })
    }
  }

  for (const line of lines) {
    if (line.trim() === '!scene') {
      flushScene()
      currentLines = []
      inScene = true
    } else if (inScene) {
      currentLines.push(line)
    }
  }

  flushScene()

  return scenes
}
```

- [ ] **Step 2: Rule 1 테스트 통과 확인**

```bash
cd front
pnpm exec vitest run src/domain/parser/parseScenes.test.ts
```

Expected: Rule 1 8개 PASS. Rule 2/3은 대상 파일이 없어 vacuously pass (0 assertions).

- [ ] **Step 3: 커밋 & 푸시**

```bash
git add front/src/domain/parser/parseScenes.ts
git commit -m "feat: parseScenes 구현"
git push
```

---

## Chunk 3: UI 레이어

### Task 5: Zustand 스토어

**Files:**
- Create: `front/src/features/editor/store.ts`

- [ ] **Step 1: 스토어 생성**

`front/src/features/editor/store.ts`:
```typescript
import { create } from 'zustand'
import { Scene } from '../../domain/scene/types'
import { parseScenes } from '../../domain/parser/parseScenes'

interface EditorStore {
  markdown: string
  scenes: Scene[]
  setMarkdown: (md: string) => void
}

export const useEditorStore = create<EditorStore>((set) => ({
  markdown: '',
  scenes: [],
  setMarkdown: (md) => set({ markdown: md, scenes: parseScenes(md) }),
}))
```

- [ ] **Step 2: 커밋 & 푸시**

```bash
git add front/src/features/editor/store.ts
git commit -m "feat: Zustand 에디터 스토어 생성"
git push
```

---

### Task 6: TextScene 컴포넌트

**Files:**
- Create: `front/src/features/editor/components/TextScene.tsx`

- [ ] **Step 1: TextScene 생성**

`front/src/features/editor/components/TextScene.tsx`:
```tsx
'use client'

import { Scene } from '../../../domain/scene/types'

interface Props {
  scene: Scene
}

export default function TextScene({ scene }: Props) {
  return (
    <div
      style={{ aspectRatio: '16 / 9' }}
      className="w-full bg-black flex items-center justify-center"
    >
      <p className="text-white text-xl text-center whitespace-pre-wrap px-8">
        {scene.content}
      </p>
    </div>
  )
}
```

- [ ] **Step 2: 커밋 & 푸시**

```bash
git add front/src/features/editor/components/TextScene.tsx
git commit -m "feat: TextScene 컴포넌트 구현"
git push
```

---

### Task 7: EditorPane 컴포넌트

**Files:**
- Create: `front/src/features/editor/components/EditorPane.tsx`

- [ ] **Step 1: EditorPane 생성**

`front/src/features/editor/components/EditorPane.tsx`:
```tsx
'use client'

import { useEditorStore } from '../store'

export default function EditorPane() {
  const { markdown, setMarkdown } = useEditorStore()

  return (
    <textarea
      className="w-full h-full resize-none p-4 font-mono text-sm bg-zinc-900 text-zinc-100 focus:outline-none"
      value={markdown}
      onChange={(e) => setMarkdown(e.target.value)}
      placeholder={'!scene\n안녕하세요\n\n!scene\n두 번째 씬'}
    />
  )
}
```

- [ ] **Step 2: 커밋 & 푸시**

```bash
git add front/src/features/editor/components/EditorPane.tsx
git commit -m "feat: EditorPane 컴포넌트 구현"
git push
```

---

### Task 8: PreviewPane 컴포넌트

**Files:**
- Create: `front/src/features/editor/components/PreviewPane.tsx`

- [ ] **Step 1: PreviewPane 생성**

`front/src/features/editor/components/PreviewPane.tsx`:
```tsx
'use client'

import { useEditorStore } from '../store'
import TextScene from './TextScene'

export default function PreviewPane() {
  const { scenes } = useEditorStore()

  if (scenes.length === 0) {
    return (
      <div className="w-full h-full flex items-center justify-center text-zinc-400 text-sm">
        마크다운을 입력하세요
      </div>
    )
  }

  return (
    <div className="w-full h-full overflow-y-auto flex flex-col gap-4 p-4 bg-zinc-800">
      {scenes.map((scene) => (
        <TextScene key={scene.id} scene={scene} />
      ))}
    </div>
  )
}
```

- [ ] **Step 2: 커밋 & 푸시**

```bash
git add front/src/features/editor/components/PreviewPane.tsx
git commit -m "feat: PreviewPane 컴포넌트 구현"
git push
```

---

### Task 9: page.tsx 레이아웃 교체

**Files:**
- Modify: `front/src/app/page.tsx`

- [ ] **Step 1: page.tsx 전체 교체**

`front/src/app/page.tsx`:
```tsx
import EditorPane from '../features/editor/components/EditorPane'
import PreviewPane from '../features/editor/components/PreviewPane'

export default function Home() {
  return (
    <div className="flex h-screen w-screen overflow-hidden">
      <div className="w-1/2 h-full">
        <EditorPane />
      </div>
      <div className="w-1/2 h-full border-l border-zinc-700">
        <PreviewPane />
      </div>
    </div>
  )
}
```

- [ ] **Step 2: 전체 Guard 테스트 통과 확인**

```bash
cd front
pnpm exec vitest run src/domain/parser/parseScenes.test.ts
```

Expected: 전체 15개 테스트 PASS
- Rule 1: 8개 (파서 동작)
- Rule 2: 4개 (Scene 타입 import 경계)
- Rule 3: 3개 (컴포넌트 파싱 금지)

- [ ] **Step 3: 개발 서버 동작 확인**

```bash
cd front
pnpm dev
```

브라우저 `http://localhost:3000`:
- 좌우 분할 화면 확인
- 왼쪽에 `!scene\n안녕하세요` 입력 → 오른쪽에 검정 배경 씬 표시
- `!scene\n씬1\n\n!scene\n씬2` 입력 → 씬 2개 세로 스택

- [ ] **Step 4: 커밋 & 푸시**

```bash
git add front/src/app/page.tsx
git commit -m "feat: 좌우 분할 에디터 레이아웃 구현"
git push
```

---

## 완료 체크리스트

- [ ] `pnpm exec vitest run` 전체 통과 (15개 테스트)
- [ ] `pnpm dev` 실행 후 브라우저에서 좌우 분할 화면 확인
- [ ] `!scene` 입력 시 미리보기 씬 실시간 렌더링 확인
- [ ] TypeScript 에러 없음 (`pnpm build` 통과)
