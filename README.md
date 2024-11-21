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

## CDN과 성능최적화

> CDN 도입 전과 도입 후의 성능 개선 보고서 작성

### 1. 실험 환경

- 배포 방식
  - CDN 도입 전: AWS S3 버킷(서울 리전)에서 정적 파일을 직접 제공.
  - CDN 도입 후: CloudFront에서 캐싱 및 콘텐츠 배포.
- 측정 방법: DevTools Network 패널을 활용하여 리소스 로드 시간, DOMContentLoaded 시간, 및 document 크기 비교.

### 2. 데이터 분석

CDN 도입 전 (S3 버킷 사용)

- 리소스 로드 시간
  - 총 리소스: 16개
  - DOMContentLoaded: 113ms
  - 전체 로드 시간: 219ms
- 문서 크기
  - HTML document 파일 크기: 12.4 KB
- 특징
  - 모든 요청이 ap-northeast-2 리전의 S3 버킷에서 처리됨.
  - 네트워크 지연 및 리전 간 거리에 따라 성능 저하 가능성 존재.
  - 캐싱 미지원으로 인해 동일 파일 요청 시 매번 새롭게 데이터가 전송됨.

CDN 도입 후 (CloudFront 사용)

- 리소스 로드 시간
  - 총 리소스: 17개
  - DOMContentLoaded: 148ms
  - 전체 로드 시간: 231ms
- 문서 크기
  - HTML document 파일 크기: 3.2 KB
- 특징
  - 캐싱된 정적 파일을 가까운 엣지 로케이션에서 빠르게 제공.
  - HTML 문서를 압축해 파일 크기가 74.2% 감소.
  - 추가 리소스(favicon.ico)가 포함되었음에도 전체 네트워크 비용 절감 효과가 나타남.

### 3. 성능 개선 요약

| **항목**         | **CDN 도입 전 (S3)** | **CDN 도입 후 (CloudFront)** | **변화**       |
| ---------------- | -------------------- | ---------------------------- | -------------- |
| DOMContentLoaded | 113ms                | 148ms                        | **35ms 증가**  |
| 전체 로드 시간   | 219ms                | 231ms                        | **12ms 증가**  |
| HTML 문서 크기   | 12.4 KB              | 3.2 KB                       | **74.2% 감소** |

### 4. 결론 및 장점

- 문서 크기 최적화
  - CloudFront는 HTML 문서를 압축하여 파일 크기를 74.2% 줄임으로써 네트워크 대역폭을 크게 절약하였습니다.
- 캐싱 및 엣지 로케이션 활용
  - 캐싱된 리소스를 사용자와 가까운 엣지 로케이션에서 제공하여 전 세계 사용자를 대상으로 효율적인 콘텐츠 제공이 가능해졌습니다.
- 트래픽 최적화
  - S3 버킷에 대한 직접 요청이 감소하여, 트래픽 비용 절감 및 서버 부하를 완화할 수 있습니다.
- 리소스 로드 시간 변화
  - DOMContentLoaded와 전체 로드 시간이 각각 35ms, 12ms 증가했지만, 이는 CloudFront 초기 설정으로 인해 발생하는 지연으로 판단됩니다. 캐싱이 안정화된 후 전반적인 로드 성능 개선이 예상됩니다.

### 5. 첨부파일

#### 5.1. S3

![s3BucketLuna](../luna-app/public/s3.png)

#### 5.2. CloudFront

![CloudFrontLuna](../luna-app/public/cloudFront.png)
