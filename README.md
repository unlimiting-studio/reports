# reports

일회성 보고서(PRD, 회고, 분석 자료 등)를 정적 사이트로 호스팅하는 Cloudflare Worker.

배포 URL: <https://reports.yechanny.workers.dev/>

## 구조

```
public/
├── index.html              ← 보고서 목록(루트 페이지)
├── 404.html                ← 커스텀 404
└── {slug}/                 ← 보고서 한 건당 폴더 하나
    ├── index.html          ← 사람이 읽는 view (시각화 포함)
    ├── PRD.md              ← 원본 마크다운 (또는 다른 source 파일)
    └── assets/             ← 보고서 전용 자산(선택)
```

`/{slug}/index.html`을 두면 `https://reports.yechanny.workers.dev/{slug}/`로 자동 노출된다.

## 새 보고서 추가하기

1. `public/{slug}/` 폴더를 만든다. slug는 kebab-case 권장 (예: `sena-v3-prd`, `2026-05-soniox-retro`).
2. 그 안에 `index.html`을 둔다. 같이 호스팅할 원본 문서(`PRD.md`, `analysis.csv` 등)도 같은 폴더에 넣으면 다운로드 가능한 URL이 자동으로 생긴다.
3. `public/index.html`(보고서 목록)에 새 카드를 한 줄 추가한다.
4. `pnpm deploy` (또는 `npx wrangler deploy`).

별도의 빌드 파이프라인은 없다. HTML/CSS/JS는 직접 인라인하거나 CDN을 써서 작성한다 (mermaid.js, Tailwind CDN 등).

## 운영 원칙

- **HTML 인라이닝 금지.** 모든 자산은 파일로 분리해서 `public/` 아래에 둔다. 4/8 devmento-slides 사건 교훈.
- **보고서 폴더는 self-contained.** 보고서별 자산은 `public/{slug}/assets/`에 두고, 다른 보고서끼리 자산을 공유하지 않는다 (의존 끊기).
- **계정/비밀 정보는 커밋 금지.** Worker 자체는 secret이 필요 없는 정적 호스팅이지만, 보고서 본문에 토큰·키·내부 URL이 들어가지 않도록 마지막에 한 번 검수한다.

## 배포

```bash
pnpm install
pnpm deploy
```

처음 배포할 때 wrangler가 Cloudflare 계정 인증을 요구한다 (`wrangler login` 또는 `CLOUDFLARE_API_TOKEN` 환경변수).

## 다른 프로젝트에서 같은 패턴을 쓰고 싶다면

이 리포 자체가 템플릿이다. 새 정적 보고서 호스팅이 필요하면 이 리포를 그대로 fork하고:

1. `wrangler.toml`의 `name`을 새 worker 이름으로 바꾼다.
2. `public/index.html`을 자기 프로젝트에 맞게 갈아끼운다.
3. 첫 보고서 폴더만 남기고 나머지는 비운다.

핵심은 `[assets]` 블록 한 덩어리뿐이다. main 엔트리 없이 정적 자산만 서빙하면 worker 코드 자체가 없어 유지보수 비용이 0에 가깝다.
