# blog

vulcanos 의 ComfyUI · Godot · 게임 개발 기록. Obsidian 볼트를 Quartz 로 정적 사이트화.

- Live: <https://vvulcanos.github.io/>
- SSG: [Quartz v4](https://quartz.jzhao.xyz)
- 편집: Obsidian (보관함 = 이 레포 루트)

## 개발

```bash
npm install
npx quartz build --serve   # 로컬 프리뷰 http://localhost:8080
```

## 배포

`main` 브랜치에 push 하면 GitHub Actions 가 빌드 후 Pages 로 배포. 수동 실행은 Actions 탭에서 workflow_dispatch.

## 구조

```
content/
├── index.md
├── posts/
│   ├── comfyui/       # 노드 기반 이미지 생성
│   ├── godot/         # 엔진 사용기 (준비 중)
│   └── game-dev/      # 파이프라인·디자인·릴리즈 (준비 중)
└── _drafts/           # 공개 전 초안 (Quartz 빌드 제외, git 추적 O)
docs/                  # 스타일 가이드·설계 스펙·구현 플랜
.github/workflows/     # 배포 파이프라인
```

`content/진행사항/` 은 내부 세션 로그 디렉터리로 `.gitignore` 로 제외.

## 포스트 작성

[`docs/POST_STYLE.md`](docs/POST_STYLE.md) 의 매뉴얼/README 톤 규약과 4섹션 골격을 따른다. 초안은 `content/_drafts/_TEMPLATE.md` 를 복사해서 시작. 자세한 워크플로는 스타일 가이드 참고.
