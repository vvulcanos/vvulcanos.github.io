# Blog 이전 · 재구축 설계

**작성일**: 2026-04-22
**대상 repo (현재)**: `vvulcanos/comfui-blog`
**대상 repo (이후)**: `vvulcanos/vvulcanos.github.io`
**로컬 경로 (현재)**: `C:\Users\vulca\Documents\comfui_obsidian\comfui\`
**로컬 경로 (이후)**: `C:\Users\vulca\Documents\blog\`

## 배경·목표

`comfui_obsidian` 볼트를 ComfyUI 전용에서 **ComfyUI · Godot · 게임 개발 기록**을 담는 범용 블로그로 확장한다. 기존 repo·Pages 는 폐기하고 새 user site(`vvulcanos.github.io`)로 이전한다. 기존 ComfyUI 포스트 5편은 인수인계/매뉴얼 성격으로 전면 재작성한다 — 단, 재작성은 이 설계에 포함된 **후속 세션**에서 수행한다.

## 범위

**이 설계가 정의하는 것**
- 로컬 폴더 평탄화 + 이름 변경
- GitHub repo 교체 + 히스토리 유지
- Quartz 설정 변경 (`baseUrl`, 콘텐츠 구조)
- 기존 공개 포스트의 드래프트화
- 배포 검증·롤백 플레이북
- 구 repo 하드 삭제 절차
- 후속 세션이 사용할 핸드오버 포스트 템플릿 (참조용)

**이 설계가 정의하지 않는 것 (후속)**
- 4편 포스트의 실제 재작성
- 커스텀 도메인(`blog.vulcanos.dev`) 연결
- Godot·game-dev 카테고리 포스트

## 확정된 결정 사항

| 영역 | 결정 |
|---|---|
| 폴더 구조 | **평탄화**. `Documents\blog\` 하나로. 볼트 + Quartz + git repo 일치 |
| 신규 repo | `vvulcanos.github.io` (user site, Public) |
| 커스텀 도메인 | 이번 세션엔 미적용. 추후 `blog.vulcanos.dev` 예정 |
| 카테고리 | `comfyui` / `godot` / `game-dev` (세 축) |
| 사이트 정체성 | 제목 `"vulcanos — notes"` 유지, 설명만 확장 |
| 재작성 톤 | 매뉴얼/README 스타일 — 소개 / 사용 이유 / 구축 방법 / 트러블슈팅 |
| 재작성 대상 포스트 | ComfyUI 세트 4편 (WF2/3 제외, 콘텐츠 수정 중) |
| 재작성 주체 | Claude 가 1편 초안 → 승인 후 나머지 결정 |
| 구 repo 처리 | Phase D(배포 검증) 모두 통과 후 **하드 삭제** |
| Git 히스토리 | **유지** (origin 만 교체) |
| 커밋 전략 | 마이그레이션 전체를 **하나의 커밋**으로 |

## 아키텍처 — Phase 순서

```
Phase A  파일시스템 이전 (로컬)
  A1. Obsidian·로컬 dev 서버 종료
  A2. mv Documents\comfui_obsidian\comfui\*  →  Documents\blog\*
  A3. rmdir Documents\comfui_obsidian\ (빈 껍데기 제거)
  A4. Obsidian 에서 신규 경로로 볼트 재오픈

Phase B  새 repo 준비
  B1. GitHub 에 vvulcanos.github.io repo 생성 (Public, 초기화 없음)
  B2. 로컬 .git origin 교체
  B3. 히스토리 유지

Phase C  설정 갱신 + 기존 글 비공개화
  C1. quartz.config.ts  baseUrl 교체
  C2. content/index.md 재작성 (3 카테고리, "공사 중")
  C3. 기존 5편 → content/_drafts/ 로 이동
  C4. README.md 재작성

Phase D  배포 검증
  D1. 로컬 npx quartz build
  D2. 로컬 프리뷰 npx quartz build --serve
  D3. push → GitHub Actions 성공 확인
  D4. https://vvulcanos.github.io/ 라이브 확인

Phase E  구 repo 하드 삭제  ← D 전부 녹색 후에만 실행
  E1. vvulcanos/comfui-blog 삭제 (gh repo delete 또는 Settings → Danger Zone)
  E2. 구 Pages 자동 종료
```

**원칙**: Phase E 는 Phase D 모두 성공한 뒤에만 실행. D 어디서든 실패하면 E 보류.

## 파일시스템 세부

### 이동 방식

같은 드라이브(C:) 내 rename → 원자적, 빠름, 파일 복사 아님.

```bash
mv "C:/Users/vulca/Documents/comfui_obsidian/comfui" "C:/Users/vulca/Documents/blog"
rmdir "C:/Users/vulca/Documents/comfui_obsidian"
```

사전 조건:
- Obsidian 종료 (볼트 잠금 해제)
- 로컬 `quartz build --serve` 종료 (있다면)
- 에디터(VSCode 등) 해당 경로 열림 해제

### 이동 후 목표 구조

```
C:\Users\vulca\Documents\blog\
├── .obsidian\                          (설정·플러그인 유지, 워크스페이스 탭은 리셋 가능)
├── .git\                               (4 commits 히스토리 유지, origin 교체)
├── .github\workflows\deploy.yml        (그대로)
├── .gitignore                          (그대로 — _drafts 이미 ignore)
├── content\
│   ├── index.md                         (재작성)
│   ├── posts\
│   │   └── comfyui\                    (빈 폴더, 재작성본은 다음 세션)
│   ├── _drafts\                        (신규)
│   │   ├── 2026-04-21-comfyui-소개-amd-zluda-설치.md
│   │   ├── 2026-04-21-custom-node-shadow-collision.md
│   │   ├── 2026-04-22-workflow-json-구조와-설계.md
│   │   ├── 2026-04-22-wf1-portrait-npc.md
│   │   └── 2026-04-22-wf2-pixel-char.md
│   ├── 진행사항\                       (gitignore, 그대로)
│   └── 환영합니다!.md                  (gitignore, 그대로)
├── quartz\  quartz.config.ts  quartz.layout.ts
├── package.json  package-lock.json  node_modules\
├── README.md                            (재작성)
└── 그 외 설정 (.prettierrc, tsconfig.json, Dockerfile 등 — 그대로)
```

### 판단 요약

- `posts/godot/`, `posts/game-dev/` 는 만들지 않는다. 빈 폴더는 Quartz `FolderPage` 가 빈 인덱스를 만들어 "없는 카테고리" 가 노출된다.
- `content/_drafts/` 는 신규 폴더. `quartz.config.ts` `ignorePatterns` 에 `_drafts` 가 이미 있음.
- 기존 5편은 전부 드래프트로 옮김. WF1 은 재작성 대상이라 공개 유지가 오히려 혼란.
- `node_modules` 는 함께 이동. 빌드 실패 시 `rm -rf node_modules && npm ci` 로 리셋.
- 내부 위키링크(`[[...]]`)는 볼트 루트 기준 상대 경로이므로 폴더명 변경에 영향 없음.

## Git · GitHub · 설정 갱신

### B1. 새 GitHub repo 생성 (사용자 액션)

github.com → New repository:
- Name: `vvulcanos.github.io`
- Public
- Initialize with README/LICENSE/.gitignore: 모두 체크 해제

`gh` CLI 옵션:
```bash
gh repo create vvulcanos/vvulcanos.github.io --public
```

### B2. origin 교체 (로컬)

```bash
cd C:/Users/vulca/Documents/blog
git remote set-url origin https://github.com/vvulcanos/vvulcanos.github.io.git
git remote -v
```

### C1. `quartz.config.ts` diff

```diff
- baseUrl: "vvulcanos.github.io/comfui-blog",
+ baseUrl: "vvulcanos.github.io",
```

그 외 필드 변경 없음.

### C2. `content/index.md` 전체 교체

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

빈 카테고리 폴더로 향하는 `[[wiki-link]]` 는 포함 안 함 — 대상이 없어 죽은 링크가 됨.

### C4. `README.md` 전체 교체

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

### 배포 워크플로

`.github/workflows/deploy.yml` — 변경 없음. repo 이름 하드코딩 없음, `actions/deploy-pages@v4` 가 컨텍스트에서 처리.

### 커밋·push

전체 변경을 하나의 커밋으로:

```bash
git add -A
git commit -m "chore: migrate to vvulcanos.github.io — flatten, retitle, move existing posts to _drafts for rewrite"
git push -u origin main
```

## 검증 · 롤백

### D1. 로컬 빌드 체크포인트

```bash
cd C:/Users/vulca/Documents/blog
npx quartz build
npx quartz build --serve   # http://localhost:8080
```

- [ ] 홈페이지 렌더 (`vulcanos — notes`, 3 카테고리 소개, "재작성 중")
- [ ] 빈 카테고리 URL (`/posts/godot/`, `/posts/game-dev/`) 은 404 — 정상
- [ ] `_drafts/` 내부 포스트 URL 직타 → 404
- [ ] `/진행사항/`, `/환영합니다!/` 미노출
- [ ] 콘솔 에러 없음

실패 시 push 금지.

### D2. CI 체크포인트

push 후 GitHub Actions:
- [ ] `build` job 녹색
- [ ] `deploy` job 녹색
- [ ] `environment: github-pages` URL 출력

### D3. 라이브 체크포인트

```bash
curl -I https://vvulcanos.github.io/   # HTTP/2 200 기대
```

- [ ] 홈페이지 로드, 제목 `vulcanos — notes`
- [ ] RSS: `/index.xml` 200
- [ ] Sitemap: `/sitemap.xml` 200
- [ ] 폰트·테마 정상
- [ ] 모바일 뷰 OK

### D4. Pages Source 설정 (최초 1회, 필수)

GitHub → `vvulcanos.github.io` repo → Settings → Pages:
- **Source: GitHub Actions** (Branch 아님)

`actions/deploy-pages@v4` 는 이 설정 없으면 권한 에러로 실패.

### 롤백 플레이북

| 실패 지점 | 조치 |
|---|---|
| D1 로컬 빌드 실패 | `baseUrl` 오타·링크 경로 확인. push 안 함 |
| D2 CI 빌드 실패 | Actions 로그 → `npm ci` 실패면 lockfile 확인. 재push |
| D2 deploy 권한 실패 | D4 Pages Source 확인 |
| D3 404 | 5~10분 CDN 대기. 지속 시 Pages 설정 재확인 |
| D3 깨진 렌더 | `git revert HEAD && git push` — 마이그레이션 커밋 1개만 revert 하면 원복 |
| 전면 롤백 | origin 을 `comfui-blog.git` 로 되돌리고 폴더 rename 역방향. 구 repo 아직 존재하므로 가능 |

**안전망**: Phase E 이전까지 `comfui-blog` repo 는 손대지 않는다. D 가 전부 녹색 되기 전엔 회복 경로가 존재.

### E. 구 repo 하드 삭제

D 전부 통과 후:

```bash
# gh CLI (확인 프롬프트 있음)
gh repo delete vvulcanos/comfui-blog --yes

# 또는 브라우저: Settings → 맨 아래 → Delete this repository → 레포 이름 타이핑
```

되돌릴 수 없음. 실행 전 마지막 확인:
- [ ] D1/D2/D3 전부 녹색
- [ ] 로컬 `blog/` 정상 작동
- [ ] 구 repo 에 저장만 해두고 싶은 내용 없음

삭제 후: `https://vvulcanos.github.io/comfui-blog/` 자동 404.

## 핸드오버 포스트 템플릿 (후속 세션 참조)

이번 세션엔 적용하지 않는다. 다음 세션에서 1편(환경 셋업) 초안 작성 시 기준선.

### 프런트매터 (공개 포스트 공통)

```yaml
---
title: "<카테고리> · <짧은 제목>"
date: YYYY-MM-DD
category: comfyui | godot | game-dev
tags: [...]
summary: "한 줄 요약 — 카드·RSS 에 노출"   # 선택, 권장
draft: false
---
```

### 본문 골격

```md
## 이 글은 무엇인가
1~2문단. 다루는 범위 + 대상 독자 + 전제.

## 왜 이게 필요한가 · 이 방법을 쓰는 이유
맥락과 대안 비교.

## 어떻게 하는가
단계적 설명 (명령어·코드·스크린샷). 중간 체크포인트. 각 단계의 *왜* 한두 줄.

## 빠지기 쉬운 함정 · 트러블슈팅
**증상 → 원인 → 해결** 형식.

## 관련
내부 포스트 위키링크, 외부 문서.
```

### 포스트별 섹션 커스터마이즈

| 포스트 | 조정 |
|---|---|
| 1. 환경 셋업 | 4섹션 그대로. "어떻게 하는가" 가 가장 길어짐 |
| 2. 커스텀 노드 가이드 | "트러블슈팅" = shadow 버그 + import 에러 패턴 |
| 3. 워크플로 설계 원칙 | "어떻게 하는가" → "설계 규약". JSON 구조 + 그룹화 + CID |
| 4. WF1 각론 | "어떻게 하는가" → "그래프 구조 분해". G0~G4 역할 + 왜 쪼갰나 |

### 톤 규칙

- 평서체 통일 (해설체 `~입니다` 금지)
- AI 티 표현 금지 ("이는 ~을 의미합니다", "매우", "다음과 같은 장점이 있습니다")
- 각 단계에 *왜* 한 줄 — 없으면 레시피가 됨, 매뉴얼이 되려면 근거 필요
- 복붙 가능한 완전한 커맨드 (예: `cd ... && npx ...`)
- 수치·경로·버전은 구체적 (예: "RX 9070 XT 16GB", "ZLUDA 3.x", "D:\AI\models")

### 다음 세션 진입 방식

1. 이 스펙 읽고 시작
2. 1편(환경 셋업) 초안 — Claude 가 `_drafts/2026-04-21-comfyui-소개-amd-zluda-설치.md` 원본 + `C:\Users\vulca\.claude\docs\comfyui-zluda.md` 참조
3. 사용자 섹션별 리뷰 → 수정 → 머지 → push (자동 배포)
4. 2편 진입 결정 (톤이 잘 잡히면 Claude 계속 드래프트, 아니면 사용자 직접)

## 리스크·주의

- **GitHub repo 하드 삭제는 되돌릴 수 없음**. 삭제 직전 D 체크리스트 재확인.
- **Pages Source 설정 누락**은 첫 배포 실패의 가장 흔한 원인. CI 가 빨갛게 뜨면 제일 먼저 확인.
- **Obsidian 워크스페이스**: 이동 후 첫 오픈 시 "last opened" 탭들이 리셋될 수 있음. 플러그인·단축키·테마 설정은 유지.
- **`node_modules` 이동 후 빌드 이슈**: 드물지만 Windows 긴 경로 관련으로 깨질 수 있음. `rm -rf node_modules && npm ci` 로 복구.
- **커스텀 도메인 전환 시**: `baseUrl` 을 `blog.vulcanos.dev` 로 바꾸고 `static/CNAME` 파일(`blog.vulcanos.dev` 한 줄) 추가. DNS 는 `CNAME blog → vvulcanos.github.io`. 이 작업은 별도 세션.
