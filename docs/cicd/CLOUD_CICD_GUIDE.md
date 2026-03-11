# 클라우드 서버 CI/CD 정리

이 문서는 외부 클라우드 서버에 이 프로젝트를 자동 배포할 때 어떤 구성이 필요한지 정리한 문서입니다.

대상:

- `config-service`
- `discovery`
- `apigateway`
- `member-service`

## 1. 전체 흐름

가장 많이 쓰는 흐름은 아래와 같습니다.

1. GitHub에 코드 푸시
2. GitHub Actions 실행
3. 테스트와 빌드 수행
4. Docker 이미지 생성
5. 이미지 레지스트리에 push
6. 클라우드 서버 또는 Kubernetes에 배포

한 줄로 보면 아래 구조입니다.

```text
개발자 -> GitHub push -> GitHub Actions -> Docker Registry -> Cloud Server(K8s)
```

## 2. 준비할 것

필요한 것은 4가지입니다.

- Git 저장소
- Docker 이미지 저장소
  - 예: Docker Hub, GHCR
- 클라우드 서버
  - 예: AWS EC2, Naver Cloud Server, GCP VM
- 배포 대상 Kubernetes 클러스터 또는 서버

## 3. 어떤 방식으로 배포하나

보통 아래 두 방식 중 하나를 고릅니다.

### 3-1. 서버에 직접 배포

구조:

1. GitHub Actions가 Docker 이미지를 push
2. 클라우드 서버에서 `docker pull`
3. 서버에서 컨테이너 재시작

이 방식은 단순합니다.

### 3-2. Kubernetes로 배포

구조:

1. GitHub Actions가 Docker 이미지를 push
2. GitHub Actions가 `kubectl apply` 또는 `kubectl set image`
3. Kubernetes가 새 Pod를 올림

지금 프로젝트는 이 방식이 더 잘 맞습니다.

이유:

- 이미 Kubernetes YAML이 있음
- 서비스가 여러 개임
- `config`, `discovery`, `apigateway`, `member-service`를 나눠 관리하기 쉬움

## 4. 가장 단순한 권장 구조

이 프로젝트에서는 아래 구조가 가장 무난합니다.

1. GitHub Actions 사용
2. 이미지 저장소는 GHCR 또는 Docker Hub 사용
3. 배포 대상은 Kubernetes 사용
4. Kubernetes에는 `msa` 네임스페이스 사용

즉:

```text
GitHub Actions
-> Docker 이미지 빌드
-> GHCR/Docker Hub push
-> kubectl로 K8s 반영
```

## 5. 저장소에 넣어둘 것

Git에 넣는 것:

- Dockerfile
- Kubernetes YAML
- 배포 스크립트
- GitHub Actions workflow 파일

Git에 넣지 않는 것:

- 실제 비밀키
- kubeconfig 원본 파일
- JWT 개인키
- `.env`

## 6. GitHub Secrets에 넣을 값

보통 아래 값을 GitHub Secrets에 넣습니다.

- `REGISTRY_USERNAME`
- `REGISTRY_PASSWORD`
- `KUBE_CONFIG`
- `TOKEN_MAKER`
- `TOKEN_PRIVATE`
- `TOKEN_PUBLIC`

설명:

- `REGISTRY_*`
  - Docker 이미지 push 용도
- `KUBE_CONFIG`
  - GitHub Actions에서 클러스터 접근할 때 사용
- `TOKEN_*`
  - `member-service` 시크릿 생성용

## 7. 배포 순서

배포 순서는 아래가 안전합니다.

1. `config-service`
2. `discovery`
3. `apigateway`
4. `member-service`

이 순서를 쓰는 이유:

- `member-service`는 `config-service`와 `discovery`를 먼저 사용함
- 마지막에 `apigateway` 기준으로 Swagger 확인 가능

## 8. CI 단계에서 하는 일

CI는 보통 여기까지입니다.

1. 코드 체크아웃
2. JDK 세팅
3. 테스트 실행
4. `jar` 빌드
5. Docker 이미지 빌드
6. 이미지 push

이 프로젝트 기준으로 보면:

- `config`, `discovery`, `apigateway`
  - `bootJar`
- `member-service`
  - `bootBuildImage` 또는 Docker 이미지 빌드

## 9. CD 단계에서 하는 일

CD는 보통 여기입니다.

1. Kubernetes 접속
2. 네임스페이스 확인
3. ConfigMap/Secret 반영
4. Deployment 이미지 교체
5. 롤아웃 확인

지금 레포 기준으로 연결할 수 있는 파일:

- [deploy-k8s.sh](/Users/parkjinwoo/source/study/grepp_BE2/MSA/scripts/deploy/deploy-k8s.sh)
- [deploy.sh](/Users/parkjinwoo/source/study/grepp_BE2/member-service/scripts/deploy.sh)

## 10. 추천 workflow 구조

파일 위치 예시:

```text
.github/workflows/deploy.yml
```

단계 예시:

1. `main` 브랜치 push 감지
2. Gradle 빌드
3. Docker 로그인
4. 이미지 build/push
5. `kubectl apply`
6. `kubectl rollout status`

## 11. 배포 전에 확인할 것

- Kubernetes 클러스터에서 이미지 저장소 접근 가능 여부
- `msa` 네임스페이스 존재 여부
- `member-service-secrets` 준비 여부
- `config-repo` ConfigMap 반영 여부

## 12. 가장 많이 막히는 부분

### 1) 이미지 pull 실패

원인:

- 이미지 push 안 됨
- 태그 불일치
- 레지스트리 로그인 문제

### 2) member-service만 기동 실패

원인:

- `TOKEN_*` 누락
- Config Server 연결 실패
- Eureka 등록 실패

### 3) gateway는 뜨는데 실제 API 호출 실패

원인:

- 서비스가 Eureka에 등록 안 됨
- 인가 체크 API가 `false` 반환
- OpenAPI `servers.url`이 게이트웨이 기준이 아님

## 13. 추천 시작 방식

처음 구성할 때는 아래처럼 나누는 것이 관리가 쉽습니다.

1. 먼저 수동 배포 성공
2. 그 다음 GitHub Actions로 빌드 자동화
3. 마지막에 Kubernetes 반영까지 자동화

즉 처음부터 모든 것을 한 번에 자동화하기보다:

```text
수동 배포 성공 -> CI 자동화 -> CD 자동화
```

이 순서가 가장 안정적입니다.
