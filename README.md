# S05p's Blog

Jekyll 기반의 개인 블로그입니다.

## 개발 환경 설정

### 1. Ruby 및 Jekyll 설치
```bash
# Ruby 설치 (macOS)
brew install ruby

# Jekyll 설치
gem install jekyll bundler
```

### 2. 의존성 설치
```bash
bundle install
```

## 로컬 개발

### 방법 1: npm 스크립트 사용 (권장)
```bash
npm run dev      # 개발 서버 실행 (livereload 포함)
npm run serve    # 개발 서버 실행
```

### 방법 2: 직접 실행
```bash
bundle exec jekyll serve --config _config.yml,_config.dev.yml --livereload
```

## 빌드

```bash
npm run build        # 로컬용 빌드
npm run build:prod   # 프로덕션용 빌드
```

## CSS 경로 문제 해결

이 프로젝트는 로컬 개발 환경과 GitHub Pages에서 모두 올바르게 작동하도록 설정되어 있습니다:

- **GitHub Pages**: `baseurl: ''` (빈 문자열)
- **로컬 개발**: `baseurl: '/'` (개발용 설정 파일 사용)

### 설정 파일 우선순위
1. `_config.yml` - 기본 설정
2. `_config.dev.yml` - 로컬 개발용 설정 (baseurl: '/')

## 폴더 구조

```
├── _config.yml          # 기본 설정
├── _config.dev.yml      # 로컬 개발용 설정
├── _includes/           # 공통 컴포넌트
├── _layouts/            # 레이아웃 템플릿
├── _posts/              # 블로그 포스트
├── public/              # 정적 파일 (CSS, 이미지 등)
└── index.html           # 메인 페이지
```

## 배포

GitHub Pages에 자동으로 배포됩니다. `main` 브랜치에 푸시하면 자동으로 빌드되어 배포됩니다.
