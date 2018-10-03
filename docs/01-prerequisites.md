# 필수사항

## Amazon Web Services

이 가이드에서는 [Amazon Web Services](https://aws.amazon.com/)를 활용하여 Kubernetes 클러스터를 처음부터 부트 스트랩하는 데 필요한 컴퓨팅 인프라 프로비저닝을 간소화합니다. 이 작업을 완료하는 데 걸리는 24 시간 동안 약 2달러 미만의 비용이 소요됩니다.

> 해당 가이드에서 필요한 컴퓨팅 리소스는 Amazon Web Services 프리 티어를 초과합니다. 원하지 않는 비용이 발생하지 않도록 학습이 끝나면 리소스를 정리하시길 바랍니다.

## Amazon Web Services CLI

### AWS CLI 설치

AWS CLI [문서](https://aws.amazon.com/cli/)를 참고하여 `aws` command line 도구를 설치해주세요.  

설치 후 아래의 명령어를 통해 설치된 버전을 확인 할 수 있습니다.

```
aws --version
```

### 기본 리전 설정

이 가이드에서는 아래와 같이 기본 리전이 설정되어 있다는 가정하에 진행됩니다.

아래의 명령어처럼 설정해주세요.

```bash
AWS_REGION=ap-northeast-2
aws configure set default.region $AWS_REGION
```

다음: [Client 도구들 설치](02-client-tools.md)
