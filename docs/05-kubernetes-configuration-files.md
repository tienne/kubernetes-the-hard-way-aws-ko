# Kubernetes 인증 처리를 위한 설정 파일 생성

이번에는 Kubeconfigs 라고 불리는 [Kubernetes 구성 파일](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)을 생성하여 Kubernetes 클라이언트가 Kubernetes API 서버를 찾아 인증이 가능하도록 진행합니다.

## 클라이언트 인증 설정

`controller manager`,`kubelet`,`kube-proxy`,`scheduler` 클라이언트와 `admin` 사용자 용 kubeconfig 파일을 생성 하도록 하겠습니다.

### Kubernetes Public DNS Address

고가용성을 지원하기 위하여 Kubernetes API Servers 에 연결된 외부 로드 밸런서에 할당된 IP 주소를 사용하여, 각 kubeconfig 에 쿠버네티스 API 서버가 연결되어 있어야합니다.   

이전 단계에서 등록한 `kubernetes-the-hard-way` DNS address를 찾습니다.

```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[0].DNSName')
```

### kubelet 관련 쿠버네티스 설정 파일

Kublet에 대한 kubeconfig 파일을 생성할 때 Kubelet의 Node 이름과 일치하는 클라이언트 인증서를 사용해야 합니다. 이를 통해 Kubernetes가 [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/) 를 통해 Kubelet을 판단하여 승인합니다.
 
각 Worker 노드에 해당하는 kubeconfig 파일을 생성합니다. 

```
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

실행결과:

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### kube-proxy 관련 쿠버네티스 설정 파일

`kube-proxy` 서비스를 위한 kubeconfig 파일을 생성합니다.

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

실행결과:

```
kube-proxy.kubeconfig
```

### kube-controller-manager 관련 쿠버네티스 설정 파일

`kube-controller-manager` 서비스를 위한 kubeconfig 파일을 생성합니다.

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

실행결과:

```
kube-controller-manager.kubeconfig
```


### kube-scheduler 관련 쿠버네티스 설정 파일

`kube-scheduler` 서비스를 위한 kubeconfig 파일을 생성합니다.

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

실행결과:

```
kube-scheduler.kubeconfig
```

### admin 관련 쿠버네티스 설정 파일

`admin` 유저를 위한 kubeconfig 파일을 생성합니다.

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

실행 결과:

```
admin.kubeconfig
```


## 

## 쿠버네티스 설정 파일 배포

`kubelet` 과 `kube-proxy` kubeconfig 파일들을 Worker 노드 인스턴스에 복사하여 넣어둡니다. 

```
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa \
    ${instance}.kubeconfig kube-proxy.kubeconfig ubuntu@${external_ip}:~/
done
```

`kube-controller-manager` 와 `kube-scheduler` kubeconfig 파일들을 Master 노드 인스턴스에 복사하여 넣어둡니다.

```
for instance in master-0 master-1 master-2; do
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  
  scp -i kubernetes.id_rsa \
    admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ubuntu@${external_ip}:~/
done
```

다음: [데이터 암호화 설정 및 키 생성](06-data-encryption-keys.md)
