# Blog 이전·재구축 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `comfui_obsidian\comfui\` 볼트를 `blog\`로 평탄화하고 GitHub user site `vvulcanos.github.io`로 이전, 기존 포스트는 `_drafts/`로 비공개화, 구 repo 하드 삭제.

**Architecture:** 같은 드라이브 내 `mv` 로 파일시스템 이전 → `git remote set-url` 로 origin 교체 → Quartz 설정·콘텐츠 갱신 → 단일 마이그레이션 커밋 → push 배포 → D 검증 전부 녹색 뒤에만 구 repo 하드 삭제.

**Tech Stack:** Obsidian vault, Quartz v4 SSG, GitHub Pages (user site via Actions), Git Bash on Windows, `gh` CLI v2.83.

**Spec:** `docs/superpowers/specs/2026-04-22-blog-migration-design.md`

**Scope exclusion:** 4편 포스트 재작성은 이 계획에 **포함되지 않음**. 후속 세션 대상.

---

## 파일 구조

**이동:**
- `C:/Users/vulca/Documents/comfui_obsidian/comfui/*` → `C:/Users/vulca/Documents/blog/*` (전체 rename)
- `content/posts/comfyui/*.md` (5편) → `content/_drafts/*.md`

**수정 (이동 후 경로):**
- `C:/Users/vulca/Documents/blog/quartz.config.ts` — `baseUrl` 1줄
- `C:/Users/vulca/Documents/blog/content/index.md` — 전체 교체
- `C:/Users/vulca/Documents/blog/README.md` — 전체 교체

**생성:**
- `C:/Users/vulca/Documents/blog/content/_drafts/` — 신규 디렉터리 (포스트 5편 이동 대상)

**Git metadata (파일 변경 아님):**
- `origin` URL 교체: `comfui-blog.git` → `vvulcanos.github.io.git`

**사용자 수동 액션:**
1. Obsidian·로컬 dev 서버 종료 (Task 1)
2. Obsidian 볼트 신규 경로로 재오픈 (Task 4)
3. GitHub 에 `vvulcanos.github.io` repo 생성 (Task 5)
4. Pages Source = GitHub Actions 설정 (Task 15)
5. 구 `comfui-blog` repo 하드 삭제 최종 확정 (Task 19)

---

## Phase A — 파일시스템 이전

### Task 1: 사전 조건 확인 (사용자 액션)

**파일:** 없음 (환경 체크)

- [ ] **Step 1: Obsidian 종료**

사용자가 Obsidian 을 직접 종료. 볼트 파일 잠금이 걸려 있으면 `mv` 가 일부 파일에서 실패할 수 있음.

- [ ] **Step 2: 로컬 dev 서버 종료**

`npx quartz build --serve` 가 실행 중이면 Ctrl+C. `.quartz-cache/` 파일 잠금 방지.

- [ ] **Step 3: 에디터 해당 경로 열림 해제**

VSCode·WebStorm 등에서 `comfui_obsidian\comfui\` 열려 있으면 닫기.

- [ ] **Step 4: 타겟 경로 비어있는지 확인**

```bash
ls "C:/Users/vulca/Documents/blog" 2>&1
```

Expected: `No such file or directory` (또는 빈 폴더). 기존 내용이 있으면 충돌하므로 중단하고 사용자 확인.

- [ ] **Step 5: 작업트리 클린 확인**

```bash
cd "C:/Users/vulca/Documents/comfui_obsidian/comfui"
git status --short
```

Expected: 출력 없음. 미커밋 변경 있으면 스태시 또는 커밋.

### Task 2: 폴더 이동 (평탄화)

**파일:** 
- Move: `C:/Users/vulca/Documents/comfui_obsidian/comfui/` → `C:/Users/vulca/Documents/blog/`

- [ ] **Step 1: 이동 실행**

```bash
mv "C:/Users/vulca/Documents/comfui_obsidian/comfui" "C:/Users/vulca/Documents/blog"
```

같은 드라이브(C:) 내 rename 이라 원자적. 즉시 완료.

- [ ] **Step 2: 이동 결과 확인**

```bash
ls "C:/Users/vulca/Documents/blog" | head -20
```

Expected: `.git`, `.github`, `.obsidian`, `content`, `quartz`, `package.json` 등 표시됨.

- [ ] **Step 3: git 루트 인식 확인**

```bash
cd "C:/Users/vulca/Documents/blog" && git rev-parse --show-toplevel
```

Expected: `C:/Users/vulca/Documents/blog`

### Task 3: 빈 래퍼 제거

**파일:**
- Delete: `C:/Users/vulca/Documents/comfui_obsidian/`

- [ ] **Step 1: 비어있는지 확인**

```bash
ls -la "C:/Users/vulca/Documents/comfui_obsidian/" 2>&1
```

Expected: `.` 과 `..` 만 있음 (또는 `No such file or directory`).

- [ ] **Step 2: 제거**

```bash
rmdir "C:/Users/vulca/Documents/comfui_obsidian" 2>&1
```

비어있지 않다고 하면 `ls -la` 로 뭐가 남았는지 확인하고 중단. 예상치 못한 파일이면 사용자에게 확인.

### Task 4: Obsidian 볼트 재오픈 (사용자 액션)

**파일:** 없음 (Obsidian 런처 작업)

- [ ] **Step 1: Obsidian 실행 → Open folder as vault**

"Open folder as vault" 에서 `C:\Users\vulca\Documents\blog` 선택. 구 볼트 목록에 남아있는 `comfui` 엔트리는 지움(경로 무효).

- [ ] **Step 2: 열림 확인**

`.obsidian/` 폴더가 자동 인식되어 기존 설정·테마·플러그인·단축키 그대로 유지. 마지막 열린 탭은 리셋될 수 있음.

---

## Phase B — 새 GitHub repo 준비

### Task 5: GitHub 에 repo 생성 (사용자 액션 또는 gh CLI)

**파일:** 없음 (GitHub 메타 작업)

- [ ] **Step 1: 방법 1 — `gh` CLI (권장)**

```bash
gh repo create vvulcanos/vvulcanos.github.io --public --description "vulcanos — notes. ComfyUI · Godot · 게임 개발 기록"
```

Expected: `✓ Created repository vvulcanos/vvulcanos.github.io on GitHub`

- [ ] **Step 1 (대안): 방법 2 — 브라우저 수동**

`github.com` → **New repository**:
- Owner: `vvulcanos`
- Repository name: `vvulcanos.github.io` (정확히 이 문자열)
- Visibility: **Public**
- Initialize: README/LICENSE/.gitignore **모두 체크 해제**
- **Create repository**

- [ ] **Step 2: 생성 확인**

```bash
gh repo view vvulcanos/vvulcanos.github.io --json name,isPrivate,isEmpty
```

Expected: `{"name":"vvulcanos.github.io","isPrivate":false,"isEmpty":true}`

### Task 6: 로컬 git origin 교체

**파일:** `C:/Users/vulca/Documents/blog/.git/config`

- [ ] **Step 1: 현재 origin 확인**

```bash
cd "C:/Users/vulca/Documents/blog"
git remote -v
```

Expected:
```
origin	https://github.com/vvulcanos/comfui-blog.git (fetch)
origin	https://github.com/vvulcanos/comfui-blog.git (push)
```

- [ ] **Step 2: origin URL 교체**

```bash
cd "C:/Users/vulca/Documents/blog"
git remote set-url origin https://github.com/vvulcanos/vvulcanos.github.io.git
```

- [ ] **Step 3: 교체 확인**

```bash
git remote -v
```

Expected:
```
origin	https://github.com/vvulcanos/vvulcanos.github.io.git (fetch)
origin	https://github.com/vvulcanos/vvulcanos.github.io.git (push)
```

---

## Phase C — 설정 갱신 + 기존 글 비공개화

### Task 7: `quartz.config.ts` baseUrl 변경

**파일:** `C:/Users/vulca/Documents/blog/quartz.config.ts` (19번째 줄)

- [ ] **Step 1: 현재 값 확인**

```bash
cd "C:/Users/vulca/Documents/blog"
grep -n "baseUrl" quartz.config.ts
```

Expected:
```
19:    baseUrl: "vvulcanos.github.io/comfui-blog",
```

- [ ] **Step 2: 값 교체**

`quartz.config.ts` 에서 정확히 한 줄 변경:

```diff
-    baseUrl: "vvulcanos.github.io/comfui-blog",
+    baseUrl: "vvulcanos.github.io",
```

- [ ] **Step 3: 변경 확인**

```bash
grep -n "baseUrl" quartz.config.ts
```

Expected:
```
19:    baseUrl: "vvulcanos.github.io",
```

### Task 8: `content/index.md` 전체 재작성

**파일:** `C:/Users/vulca/Documents/blog/content/index.md`

- [ ] **Step 1: 기존 파일 전체를 아래 내용으로 교체**

```md
---
title: "vulcanos — notes"
---

# vulcanos — notes

ComfyUI · Godot · 게임 개발 기록. 만들면서 부딪힌 문제와 해결 과정을 매뉴얼처럼 정리.

## 카테고리

- **ComfyUI** — 노드 기반 이미지 생성 UI. AMD ZLUDA 환경, 워크플로 설계, 트러블슈팅.
- **Godot** — 엔진 사용기. (준비 중)
- **Game Dev** — 파이프라인·디자인·릴리즈 회고. (준비 중)

## 최근 글

_재작성 중. 곧 업데이트._
```

- [ ] **Step 2: 내용 확인**

```bash
cd "C:/Users/vulca/Documents/blog"
head -20 content/index.md
```

Expected: 위 내용 그대로.

### Task 9: 기존 포스트 5편을 `_drafts/` 로 이동

**파일:**
- Create dir: `C:/Users/vulca/Documents/blog/content/_drafts/`
- Move: `content/posts/comfyui/*.md` (5편) → `content/_drafts/`

- [ ] **Step 1: `_drafts/` 디렉터리 생성**

```bash
cd "C:/Users/vulca/Documents/blog"
mkdir -p content/_drafts
```

- [ ] **Step 2: 이동할 파일 확인**

```bash
ls content/posts/comfyui/
```

Expected: 아래 5개:
```
2026-04-21-comfyui-소개-amd-zluda-설치.md
2026-04-21-custom-node-shadow-collision.md
2026-04-22-wf1-portrait-npc.md
2026-04-22-wf2-pixel-char.md
2026-04-22-workflow-json-구조와-설계.md
```

- [ ] **Step 3: git mv 로 이동 (히스토리 보존)**

```bash
cd "C:/Users/vulca/Documents/blog"
git mv content/posts/comfyui/2026-04-21-comfyui-소개-amd-zluda-설치.md content/_drafts/
git mv content/posts/comfyui/2026-04-21-custom-node-shadow-collision.md content/_drafts/
git mv content/posts/comfyui/2026-04-22-wf1-portrait-npc.md content/_drafts/
git mv content/posts/comfyui/2026-04-22-wf2-pixel-char.md content/_drafts/
git mv content/posts/comfyui/2026-04-22-workflow-json-구조와-설계.md content/_drafts/
```

- [ ] **Step 4: 이동 결과 확인**

```bash
ls content/_drafts/
ls content/posts/comfyui/ 2>&1
```

Expected:
- `content/_drafts/` 에 5편 존재
- `content/posts/comfyui/` 는 빈 폴더 (또는 존재하면 비어있음)

- [ ] **Step 5: 빈 `content/posts/comfyui/` 폴더는 유지**

재작성본이 들어올 자리로 보존. 빈 폴더라도 `.gitkeep` 같은 건 불필요 (git 은 빈 폴더를 추적 안 함 — 재작성 시 첫 포스트가 들어가면서 폴더가 다시 등장).

### Task 10: `README.md` 전체 재작성

**파일:** `C:/Users/vulca/Documents/blog/README.md`

- [ ] **Step 1: 기존 파일 전체를 아래 내용으로 교체**

````md
# blog

vulcanos 의 ComfyUI · Godot · 게임 개발 기록. Obsidian 볼트를 Quartz 로 정적 사이트화.

- Live: <https://vvulcanos.github.io/>
- SSG: [Quartz v4](https://quartz.jzhao.xyz)
- 편집: Obsidian (보관함 = 이 레포 루트)

## 개발

```bash
npm install
npx quartz build --serve   # http://localhost:8080
```

## 배포

`main` 브랜치에 push 하면 GitHub Actions 가 빌드 후 Pages 로 배포.

## 구조

```
content/
├── index.md
├── posts/
│   ├── comfyui/       # 노드 기반 이미지 생성
│   ├── godot/         # 엔진 사용기 (준비 중)
│   └── game-dev/      # 파이프라인·디자인·릴리즈 (준비 중)
└── _drafts/           # 공개 전 초안 (Quartz 빌드 제외)
```

`content/진행사항/` 은 내부 세션 로그 디렉터리로 `.gitignore` 로 제외.
````

- [ ] **Step 2: 내용 확인**

```bash
cd "C:/Users/vulca/Documents/blog"
head -5 README.md
```

Expected 첫 줄: `# blog`, 두 번째 비어있는 줄 이후 "vulcanos 의 ComfyUI · Godot · 게임 개발 기록..." 으로 시작.

---

## Phase D — 검증 · 커밋 · push

### Task 11: 로컬 빌드 검증

**파일:** 없음 (빌드 실행)

- [ ] **Step 1: 빌드 시도**

```bash
cd "C:/Users/vulca/Documents/blog"
npx quartz build
```

Expected 마지막 출력: `Done processing N files`, `Emitted M files to \`public\` in Xs`. 에러 없음.

- [ ] **Step 2: 빌드 실패 시 복구**

`node_modules` 이동으로 인한 이슈면:

```bash
rm -rf node_modules .quartz-cache
npm ci
npx quartz build
```

- [ ] **Step 3: `public/` 출력물 확인**

```bash
ls public/
```

Expected: `index.html`, `posts/`, `index.xml`, `sitemap.xml` 등 존재.

- [ ] **Step 4: `_drafts/` 미노출 확인**

```bash
find public -name "*wf1*" -o -name "*wf2*" -o -name "*zluda*" -o -name "*workflow-json*" -o -name "*shadow-collision*" 2>&1
```

Expected: 출력 없음. 드래프트는 빌드 제외.

- [ ] **Step 5: 빈 카테고리 폴더 페이지 미생성 확인**

```bash
ls public/posts/ 2>&1
```

Expected: `comfyui/` 조차 존재하지 않거나 빈 상태. `godot/`, `game-dev/` 는 절대 없어야 함.

### Task 12: 로컬 프리뷰 수동 확인

**파일:** 없음 (브라우저 확인)

- [ ] **Step 1: dev 서버 실행**

```bash
cd "C:/Users/vulca/Documents/blog"
npx quartz build --serve
```

브라우저에서 `http://localhost:8080` 오픈.

- [ ] **Step 2: 홈페이지 체크리스트**

- [ ] 제목 `vulcanos — notes` 표시
- [ ] 3 카테고리 설명 표시 (ComfyUI / Godot / Game Dev)
- [ ] "재작성 중. 곧 업데이트." 문구 표시
- [ ] 폰트 (Schibsted Grotesk / Source Sans Pro) 로드됨
- [ ] 라이트/다크 모드 토글 동작
- [ ] 브라우저 DevTools (F12) → Console 에러 없음

- [ ] **Step 3: URL 직접 조회로 드래프트·무시 대상 미노출 확인**

브라우저 주소창에 각각 입력:
- `http://localhost:8080/posts/comfyui/2026-04-22-wf1-portrait-npc` → 404
- `http://localhost:8080/진행사항/2026-04-21-세션1-plan012-완료` → 404
- `http://localhost:8080/환영합니다!` → 404

- [ ] **Step 4: 서버 종료**

터미널에서 Ctrl+C.

### Task 13: 마이그레이션 단일 커밋

**파일:** 스테이지 전체

- [ ] **Step 1: 스테이지 확인**

```bash
cd "C:/Users/vulca/Documents/blog"
git status
```

Expected 변경 파일들:
- modified: `quartz.config.ts`
- modified: `content/index.md`
- modified: `README.md`
- renamed: `content/posts/comfyui/*.md` → `content/_drafts/*.md` (5건)

- [ ] **Step 2: 전체 스테이지**

```bash
git add -A
git status
```

Expected: 모든 변경 `Changes to be committed` 에 있음.

- [ ] **Step 3: 커밋**

```bash
git commit -m "chore: migrate to vvulcanos.github.io — flatten, retitle, move existing posts to _drafts for rewrite"
```

Expected: 1 commit created. 변경 요약에 파일 수 표시.

- [ ] **Step 4: 로그 확인**

```bash
git log --oneline -5
```

Expected (최신 → 과거):
```
<new> chore: migrate to vvulcanos.github.io — flatten, retitle, move existing posts to _drafts for rewrite
36817ae docs: blog 이전·재구축 설계 스펙 추가
<plan commit>
4392ff8 posts: AI 티 나는 말투·거창한 섹션 제목 걷어내기
...
```

### Task 14: 새 remote 로 push

**파일:** 없음 (네트워크 작업)

- [ ] **Step 1: push 실행**

```bash
cd "C:/Users/vulca/Documents/blog"
git push -u origin main
```

Expected: 커밋 히스토리 전체(5+ 커밋)가 새 repo 로 푸시됨. `Writing objects: ...`, `* [new branch]      main -> main`, `Branch 'main' set up to track 'origin/main'`.

- [ ] **Step 2: 원격 상태 확인**

```bash
gh repo view vvulcanos/vvulcanos.github.io --json defaultBranchRef,isEmpty
```

Expected: `{"defaultBranchRef":{"name":"main"},"isEmpty":false}`

### Task 15: Pages Source = GitHub Actions 설정 (사용자 액션, 최초 1회)

**파일:** 없음 (GitHub Settings UI)

- [ ] **Step 1: GitHub Pages 설정 페이지 오픈**

브라우저에서 `https://github.com/vvulcanos/vvulcanos.github.io/settings/pages` 방문.

또는 `gh` 로:
```bash
gh browse --repo vvulcanos/vvulcanos.github.io --settings
# 또는 직접 URL
start https://github.com/vvulcanos/vvulcanos.github.io/settings/pages
```

- [ ] **Step 2: Source 드롭다운을 GitHub Actions 로 변경**

**Source: GitHub Actions** 선택. "Deploy from a branch" 가 아님.

중요: 이 설정 없으면 `actions/deploy-pages@v4` 가 권한 에러로 실패.

- [ ] **Step 3: 설정 확인**

페이지에 `Your site is live at ...` 또는 `Your site is being built` 메시지 표시됨.

### Task 16: CI 빌드·배포 검증

**파일:** 없음 (GitHub Actions)

- [ ] **Step 1: 최신 워크플로 런 상태 확인**

```bash
gh run list --repo vvulcanos/vvulcanos.github.io --limit 1
```

Expected: 최신 런이 `completed success` 또는 `in_progress`.

- [ ] **Step 2: 진행 중이면 대기**

```bash
gh run watch --repo vvulcanos/vvulcanos.github.io
```

Expected 완료 시: `✓ [workflow] completed successfully`

- [ ] **Step 3: 실패 시 로그 확인**

```bash
gh run view --repo vvulcanos/vvulcanos.github.io --log-failed
```

흔한 실패:
- `npm ci` 실패 → `package-lock.json` 상태 확인
- deploy 권한 에러 → Task 15 Pages Source 재확인
- `actions/deploy-pages@v4` 실패 → repo Settings → Actions → Workflow permissions = Read and write

### Task 17: 라이브 URL 검증

**파일:** 없음 (외부 URL)

- [ ] **Step 1: HTTP 응답 확인**

```bash
curl -sI https://vvulcanos.github.io/ | head -1
```

Expected: `HTTP/2 200`

404 이면 5~10분 대기 (CDN 전파). 지속되면 Task 15 Pages Source 재확인.

- [ ] **Step 2: RSS·Sitemap 확인**

```bash
curl -sI https://vvulcanos.github.io/index.xml | head -1
curl -sI https://vvulcanos.github.io/sitemap.xml | head -1
```

Expected: 양쪽 `HTTP/2 200`.

- [ ] **Step 3: 드래프트·무시 대상 404 확인**

```bash
curl -sI https://vvulcanos.github.io/posts/comfyui/2026-04-22-wf1-portrait-npc | head -1
curl -sI "https://vvulcanos.github.io/진행사항/2026-04-21-세션1-plan012-완료" | head -1
```

Expected: 양쪽 `HTTP/2 404`.

- [ ] **Step 4: 브라우저 최종 확인**

`https://vvulcanos.github.io/` 를 브라우저로 오픈 → Task 12 체크리스트 재수행 (폰트, 테마, 모바일).

---

## Phase E — 구 repo 하드 삭제

### Task 18: 최종 사전 체크 (삭제 직전)

**파일:** 없음 (체크리스트)

- [ ] **Step 1: D 전부 녹색 확인**

- [ ] Task 11 로컬 빌드 성공
- [ ] Task 12 로컬 프리뷰 체크 OK
- [ ] Task 14 push 성공
- [ ] Task 15 Pages Source = GitHub Actions
- [ ] Task 16 CI 녹색
- [ ] Task 17 라이브 URL 200 + 콘텐츠 OK

하나라도 빨간불이면 Task 19 실행 금지. 원인 해결 후 재검증.

- [ ] **Step 2: 구 repo 에 저장만 해두고 싶은 게 있는지 확인**

```bash
gh api repos/vvulcanos/comfui-blog/issues --jq '.[].title' 2>&1
gh api repos/vvulcanos/comfui-blog/pulls --jq '.[].title' 2>&1
```

Expected: 빈 결과 또는 보존 필요 없는 것들.

- [ ] **Step 3: 사용자 재확인**

"구 repo `vvulcanos/comfui-blog` 하드 삭제를 실행합니다. 되돌릴 수 없습니다. 진행?" 에 사용자가 명시 YES 답하지 않으면 중단.

### Task 19: 구 repo 하드 삭제 (사용자 승인 후)

**파일:** 없음 (GitHub 메타 작업)

- [ ] **Step 1: 방법 1 — `gh` CLI**

```bash
gh repo delete vvulcanos/comfui-blog --yes
```

`--yes` 플래그가 있어도 `gh` 는 repo 이름 재입력 프롬프트를 띄움. 사용자가 직접 입력.

- [ ] **Step 1 (대안): 방법 2 — 브라우저 수동**

`https://github.com/vvulcanos/comfui-blog/settings` 하단 Danger Zone → **Delete this repository** → repo 이름 타이핑 → 확정.

- [ ] **Step 2: 삭제 확인**

```bash
gh repo view vvulcanos/comfui-blog 2>&1
```

Expected: `GraphQL: Could not resolve to a Repository with the name 'vvulcanos/comfui-blog'.`

- [ ] **Step 3: 구 Pages URL 404 확인**

```bash
curl -sI https://vvulcanos.github.io/comfui-blog/ | head -1
```

Expected: `HTTP/2 404`. (전파 시간 약간 걸릴 수 있음.)

### Task 20: 최종 상태 정리

**파일:** 없음 (요약)

- [ ] **Step 1: 신규 사이트 최종 확인**

```bash
curl -sI https://vvulcanos.github.io/ | head -1
```

Expected: `HTTP/2 200`

- [ ] **Step 2: 로컬 볼트 최종 확인**

```bash
cd "C:/Users/vulca/Documents/blog"
git remote -v
git status
git log --oneline -3
```

Expected:
- origin = `vvulcanos.github.io.git`
- 작업트리 클린
- 최근 커밋에 마이그레이션 커밋 + 스펙/플랜 커밋

- [ ] **Step 3: 이전 완료 보고**

이전 완료 요약:
- 폴더: `comfui_obsidian\comfui\` → `blog\`
- Repo: `comfui-blog` → `vvulcanos.github.io`
- Pages URL: `vvulcanos.github.io/comfui-blog` (삭제됨) → `vvulcanos.github.io` (새 사이트)
- 기존 5편: `_drafts/` 로 비공개화
- 다음 세션: 1편(환경 셋업) 재작성 (ComfyUI 세트 템플릿 고정)

---

## 롤백 시나리오

각 Task 가 실패했을 때의 복구:

| 실패 Task | 조치 |
|---|---|
| Task 2 mv 실패 (파일 잠김) | Task 1 재수행 (앱·서버 종료). 잠긴 파일명 확인 → 해당 앱 재시도 |
| Task 5 gh 인증 문제 | `gh auth login` 재실행 |
| Task 11 빌드 실패 | `rm -rf node_modules .quartz-cache && npm ci` 후 재빌드 |
| Task 14 push 거부 (`non-fast-forward`) | 신규 repo 가 비어있지 않다는 뜻. `gh repo view` 로 확인, 필요 시 `git push -u origin main --force` (단, 새 repo 가 정말 비어있을 때만) |
| Task 16 CI 빨간불 | Task 15 Pages Source 재확인. `gh run view --log-failed` 로 원인 파악 |
| Task 17 라이브 404 지속 | 5~10분 대기. 그래도 404 면 Settings → Pages 에서 배포 URL 확인 |
| Task 13 이후 전면 롤백 필요 | `git remote set-url origin https://github.com/vvulcanos/comfui-blog.git` → `git revert HEAD` → 폴더를 원래 경로로 역 rename. 구 repo 는 Task 19 이전까지 살아있으므로 회복 가능 |
