# 프론트엔드 배포 파이프라인

## 개요

![프론트엔드 배포 파이프라인 다이어그램](../luna-app/public/frontendDiagram.png)

### 1. **이벤트 트리거**

- **`push`**: `main` 브랜치에 코드가 푸시되면 워크플로우가 실행됩니다.
- **`workflow_dispatch`**: 수동으로 워크플로우를 실행할 수도 있습니다.

### 2. **Job: Deploy**

- **`runs-on: ubuntu-latest`**: 최신 Ubuntu 환경에서 실행됩니다.

### 3. **Steps 설명**

### 3.1 **Checkout repository**

- GitHub 저장소의 코드를 체크아웃합니다.

### 3.2 **Install dependencies**

- 프로젝트의 `package-lock.json` 파일을 기반으로 정확한 종속성을 설치합니다.

### 3.3 **Build**

- Next.js 애플리케이션을 빌드합니다.
  - 일반적으로 `out/` 디렉토리에 정적 파일이 생성됩니다.

### 3.4 **Configure AWS credentials**

- AWS에 접근하기 위해 GitHub Secrets에 저장된 AWS 자격 증명을 설정합니다.

### 3.5 **Deploy to S3**

- Next.js 빌드 아티팩트를 **S3 버킷**에 동기화합니다.
  - `-delete` 옵션: S3 버킷에서 불필요한 파일을 제거합니다.

### 3.6 **Invalidate CloudFront cache**

- **CloudFront 캐시 무효화**를 수행하여 최신 빌드가 사용자에게 즉시 제공되도록 합니다.

## 주요 링크

- S3 버킷 웹사이트 엔드포인트:
  - http://lunahanghaebucket.s3-website.ap-northeast-2.amazonaws.com
- CloudFrount 배포 도메인 이름:
  - d2q8qi9fgah1w.cloudfront.net

## 주요 개념

- GitHub Actions과 CI/CD 도구:
  - GitHub Actions는 소스 코드 변경 시 자동으로 빌드, 테스트, 배포를 실행하는 CI/CD(Continuous Integration/Continuous Deployment) 도구입니다. 파이프라인을 정의하는 YAML 파일을 통해 Next.js 앱을 자동 배포합니다.
- S3와 스토리지:
  - Amazon S3는 정적 웹사이트 호스팅과 정적 자산(HTML, CSS, JavaScript, 이미지 등)의 저장을 지원하는 클라우드 스토리지 서비스입니다. 배포된 Next.js 앱의 빌드 결과물을 업로드하는 용도로 사용됩니다.
- CloudFront와 CDN:
  - Amazon CloudFront는 전 세계의 사용자에게 웹 콘텐츠를 빠르고 안전하게 제공하는 **콘텐츠 전송 네트워크(CDN)**입니다. S3에 저장된 정적 콘텐츠를 전송하고, 요청을 가장 가까운 엣지 로케이션에서 처리합니다.
- 캐시 무효화(Cache Invalidation):
  - CloudFront는 기본적으로 콘텐츠를 캐싱하여 성능을 최적화합니다. 새로 배포된 파일이 S3에 업로드되더라도 CloudFront 캐시가 갱신되지 않으면 오래된 콘텐츠가 제공될 수 있습니다. 이를 방지하기 위해 aws cloudfront create-invalidation 명령어를 사용하여 CloudFront 캐시를 비웁니다.
- Repository secret과 환경변수:
  - GitHub Actions에서 민감한 정보를 안전하게 저장하고 사용하는 방법입니다. AWS 자격 증명(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY), S3 버킷 이름, CloudFront 배포 ID 등을 repository secret으로 설정하여 워크플로에서 참조합니다. 이를 통해 민감한 데이터 노출 없이 CI/CD 파이프라인을 실행할 수 있습니다.
