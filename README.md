# BL Check (AI) · 운영 가이드

신코르 해운 BL (Bill of Lading) 자동 체크 시스템의 운영 / 코드 / 데이터 모델 문서입니다.

## 문서 사이트

📖 **https://ysh21368ai.github.io/bl-check-ai-docs/**

## 로컬 미리보기

```bash
# 의존성 설치
pip install "mkdocs-material>=9.5" "mkdocs>=1.6" pymdown-extensions

# 로컬 서버 (http://127.0.0.1:8000)
mkdocs serve

# 빌드 (site/ 디렉토리)
mkdocs build --strict
```

## 디렉터리 구조

```
bl-check-ai-docs/
├── docs/
│   ├── index.md                            # 홈
│   ├── tech/                               # 기술 문서
│   │   ├── architecture.md
│   │   ├── data-model.md
│   │   ├── ai-pipeline.md
│   │   ├── rules.md
│   │   ├── deployment.md
│   │   └── modules/                        # 모듈 레퍼런스
│   │       ├── main.md
│   │       ├── database-handler.md
│   │       ├── oracle-store.md
│   │       ├── procedures.md
│   │       └── llm-extractor.md
│   ├── operations/                         # 운영
│   │   ├── runbook.md
│   │   ├── troubleshooting.md
│   │   └── monitoring.md
│   └── api/
│       └── README.md
├── mkdocs.yml
└── .github/workflows/docs.yml              # GitHub Pages 자동 배포
```

## 자동 배포

`docs/` 또는 `mkdocs.yml` 변경 후 `main` 브랜치에 push 하면 GitHub Actions 가 자동으로:

1. mkdocs build (`--strict`)
2. GitHub Pages 에 배포

## 본 시스템 코드 위치

코드는 별도 (사내) 저장소에서 관리. 본 repo 는 문서만.

작업 디렉토리: `/home/dev01/seunghyun/project/고객지원팀/bl_check_management_arrange - final_kr/`

## 라이선스

내부용 (Internal use only).
