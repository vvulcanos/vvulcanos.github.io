# comfui-blog

로컬 AI 이미지 파이프라인·개발 기록을 담은 디지털 가든.

- Live: <https://vvulcanos.github.io/comfui-blog/>
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
content/              # 공개 블로그 소스 (Obsidian 볼트 내부)
├── index.md
└── posts/
    └── comfyui/
quartz/               # SSG 툴링
.github/workflows/    # 배포 파이프라인
```

`content/진행사항/` 는 내부 세션 로그 디렉터리라 `.gitignore` 로 공개 레포에서 제외.
