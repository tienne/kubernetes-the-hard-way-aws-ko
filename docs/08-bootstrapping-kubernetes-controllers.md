# 쿠버네티스 마스터 노드 준비

In this lab you will bootstrap the Kubernetes control plane across three compute instances and configure it for high availability. You will also create an external load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## 준비사항

이번 단계의 명령어들은 각 마스터 인스턴스들인 `master-0`, `master-1`, `master-2` 에서 각각 실행해야합니다.
각 마스터 인스턴스에 `ssh` 로그인하기 위해 IP를 아래의 명령어를 통해 확인합니다.
 
```bash
for instance in master-0 master-1 master-2; do
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  
  echo $external_ip
done
```

이제 위의 명령어를 통해 출력된 IP를 이용하여 ssh 접속을 합니다.

```bash
ssh -i kubernetes.id_rsa ubuntu@${위에 출력된 IP1}
ssh -i kubernetes.id_rsa ubuntu@${위에 출력된 IP2}
ssh -i kubernetes.id_rsa ubuntu@${위에 출력된 IP3}
```

> 저는 3대의 인스턴스를 동시에 명령을 실행하기 위해 [iterm2](https://www.iterm2.com/) 를 이용했습니다.

## 쿠버네티스 Control Plane 구

쿠버네티스 관련 설정들을 저장할 디렉토리를 생성합니다.

```bash
sudo mkdir -p /etc/kubernetes/config
```

### Kubernetes Controller 바이너리 다운로드 및 설치

Kubernetes 공식 릴리즈 바이너리 파일들을 다운로드 합니다.

```bash
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.11.3/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.11.3/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.11.3/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.11.3/bin/linux/amd64/kubectl"
```

다운로드 받은 바이너리 파일들을 설치합니다.

```bash
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

### 쿠버네티스 API Server 설정

```bash
sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
```

인스턴스 내부 아이피 주소는 API 서버를 클러스터들에게 알려주기 위해 사용됩니다. 현재 인스턴스 내부 아이피를 가져옵니다. 

```bash
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
```

`kube-apiserver.service` ststemd 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 쿠버네티스 Controller Manager 설정

mv 명령어를 이용해서 `kube-controller-manager` kubeconfig 파일를 옴겨줍니다.

```bash
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

`kube-controller-manager.service` systemd 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 쿠버네티스 Scheduler 설정

mv 명령어를 이용해서 `kube-scheduler` kubeconfig 파일을 옴겨줍니다.

```bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

`kube-scheduler.yaml` 설정 파일을 아래와 같이 생성합니다.

```bash
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: componentconfig/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

`kube-scheduler.service` systemd 파일을 생서합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Controller Services 실행

```bash
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

> 쿠버네티스 API 서버가 초기화되는데 약 30초 정도 소요됩니다.

이제 실행 상태들을 체크합니다.

```bash
kubectl get componentstatuses
```

실행결과: 
```
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```
컨트롤러들이 위와 같이 정상적으로 실행되었습니다.

## Kubelet 인증을 위한 RBAC 

이제 Kubernetes API 서버가 각 Worker 노드 Kubelet 에 액세스 할 수 있도록 RBAC 권한을 구성하도록 하겠습니다. Pod에서 메트릭, 로그 및 실행 명령을 검색하려면 Kubelet API에 액세스해야합니다. 

> 이 가이드에서는 Kubelet `--authorization-mode` 설정을 `Webhook` 으로 설정합니다. Webhook 모드는 [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API 를 이용하여 권한설정을 합니다.

```bash
external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=controller-0" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i kubernetes.id_rsa ubuntu@${external_ip}
```

Kubelet API 에 접근 권한이 있는 `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) 을 생성을 합니다.  

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

쿠버네티스 API 서버는 `--kubelet-client-certificate` 옵션으로 정의 된 클라이언트 인증서를 사용하여 Kubelet 에 `kubernetes` 사용자로 인증합니다.

`kubernetes` 유저에 `system:kube-apiserver-to-kubelet` ClusterRole 를 적용시킵니다.

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

### 개인 PC 에서 클러스터 공개 endpoint 확인

이제 개인 노트북 혹은 데스크탑에서 아래의 명령어를 실행하여 `kubernetes-the-hard-way` 로드 밸런서 주소를 조회합니다. 

```bash
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
```

HTTP 요청으로 Kubernetes 버전 정보를 확인합니다.

```bash
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> 실행결과

```
{
  "major": "1",
  "minor": "11",
  "gitVersion": "v1.11.2",
  "gitCommit": "bb9ffb1654d4a729bb4cec18ff088eacc153c239",
  "gitTreeState": "clean",
  "buildDate": "2018-08-07T23:08:19Z",
  "goVersion": "go1.10.3",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

다음: [Kubernetes Worker 노드 준비](09-bootstrapping-kubernetes-workers.md)
