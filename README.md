# 들어가기 앞서...

이 저장소는 prabhatsharma 의 [kubernetes the hard way aws](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws) 를 번역 및 참고한 내용입니다. 또한 kelseyhightower 의 [kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way) 에서 gcp -> aws 로 변경된 가이드로 아래의 내용들이 추가 및 변경되었습니다.

1. kubernetes to v1.11.3
2. cri-tools to v1.11.1
3. containerd to v1.2.0-beta.2
4. CNI plugins to v0.7.1
5. etcd to 3.3.9
6. critools 관련 내용 추가

발번역으로 수정사항 혹은 피드백은 언제든지 환영입니다.

# Kubernetes The Hard Way

이 가이드는 kubernetes 를 자동화된 설치 방법이 아닌 하나하나 수동으로 설치하는 방법을 안내합니다. 만약 자동화된 방법으로 kubernetes 를 설치하는 방법을 알고 싶으시다면 [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine), [AWS Elastic Container Service for Kubernetes](https://aws.amazon.com/eks/) 혹은 [Getting Started Guides](http://kubernetes.io/docs/getting-started-guides/) 등을 참고하시기 바랍니다.

쿠버네티스의 부트스트랩 하기 위한 과정들을 이해하고 배우는데 최적화 되어있는 가이드입니다.

> 이 가이드의 최종 결과물은 실서비스에 적합하지 않으며, 커뮤니티에 도움이 제한되지만 해당 가이드를 중단하지 않고 끝까지 학습하는걸 추천드립니다. 

## 대상 독자

이 튜토리얼의 대상 독자는 쿠버네티스를 실 서비스에 도입할 예정이 있으며, 실서비스 도입때 어떻게 적용해야하는지 알고 싶은 모든 사람들입니다. 

## Cluster Details

이 가이드는 RBAC(Role-based access control) 인증과 컴퍼넌트간의 end-to-end(E2E) 암호화를 통해 고가용성 쿠버네티스 클러스터를 부트스트래핑하는 과정을 설명하고 있습니다.

* [Kubernetes](https://github.com/kubernetes/kubernetes) 1.11.3
* [containerd Container Runtime](https://github.com/containerd/containerd) 1.2.0-beta.2
* [gVisor](https://github.com/google/gvisor) 08879266fef3a67fac1a77f1ea133c3ac75759dd
* [CNI Container Networking](https://github.com/containernetworking/cni) 0.7.1
* [etcd](https://github.com/coreos/etcd) 3.3.9

## 목차

이 가이드는 [Amazon Web Service](https://aws.amazon.com/)에 접근 할 수 있다고 가정하고 진행합니다.
만약 GCP 버전의 가이드를 보고자 한다면 해당 가이드를 참고하세요. [https://github.com/kelseyhightower/kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

* [필수사항](docs/01-prerequisites.md)
* [Client 용 도구들 설치](docs/02-client-tools.md)
* [Provisioning Compute Resources](docs/03-compute-resources.md)
* [Provisioning the CA and Generating TLS Certificates](docs/04-certificate-authority.md)
* [Kubernetes 인증 처리를 위한 설정 파일 생성](docs/05-kubernetes-configuration-files.md)
* [데이터 암호화 설정 및 키 생성](docs/06-data-encryption-keys.md)
* [etcd Cluster 준비](docs/07-bootstrapping-etcd.md)
* [Bootstrapping the Kubernetes Control Plane](docs/08-bootstrapping-kubernetes-controllers.md)
* [Bootstrapping the Kubernetes Worker Nodes](docs/09-bootstrapping-kubernetes-workers.md)
* [Configuring kubectl for Remote Access](docs/10-configuring-kubectl.md)
* [Provisioning Pod Network Routes](docs/11-pod-network-routes.md)
* [Deploying the DNS Cluster Add-on](docs/12-dns-addon.md)
* [Smoke Test](docs/13-smoke-test.md)
* [Cleaning Up](docs/14-cleanup.md)
