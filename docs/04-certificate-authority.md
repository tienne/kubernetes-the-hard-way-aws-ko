# TLS 인증서 생성 및 인증기관(CA) 구성 

이번 단계에서는 CloudFlare의 PKI 도구인 [cfssl](https://github.com/cloudflare/cfssl) 를 이용하여 [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) 를 생성합니다.
이를 이용하여 인증기관을 bootstrap 하고, etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet 및 kube-proxy 들의 TLS 인증서를 생성합니다.

## 인증기관(Certificate Authority)

TLS 인증서 생성할때 사용될 인증기관을 생성해야합니다. 

아래의 명령어들을 통해 CA 설정 파일, 인증서, 비공개 키를 생성합니다.

```bash
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "L": "Seoul",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Seoul"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

실행결과:

```
ca-key.pem
ca.pem
```

## Client 와 Server 인증서

이번에는 Kubernetes 클라이언트 와 서버 그리고 Kubernetes `admin` 계정을 위한 인증서를 생성합니다. 

### Admin Client 인증서

`admin` 클라이언트용 인증서와 비공개 키를 생성합니다.

```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "L": "Seoul",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Seoul"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

실행결과:

```
admin-key.pem
admin.pem
```

### Kubelet Client 인증서

Kubernetes 는 Node Authorizer 라 불리는 [특수한 목적의 인증 방식](https://kubernetes.io/docs/admin/authorization/node/)를 사용하며, 이 인증 방식으로 [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet) 에서 발송하는 API 요청들을 승인합니다.
Node Authorizer 에게 권한을 부여 받기 위해 Kubelets 은 `system : nodes` 그룹에 속한 사용자인 `system : node : <nodeName>` 로 식별하는 인증서를 사용해야합니다.  
이번 단계에서는 Node Authorizer 요구 사항을 충족하는 Kubernetes 각 Worker 노드에 대한 인증서를 생성합니다.  

Kubernetes Worker 노드용 인증서 와 비공개 키 생성합니다.

```bash
for i in 0 1 2; do
  instance="worker-${i}"
  instance_hostname="ip-10-240-0-2${i}"
  cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance_hostname}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "L": "Seoul",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Seoul"
    }
  ]
}
EOF

external_ip=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=${instance}" \
  --output text --query 'Reservations[].Instances[].PublicIpAddress')

internal_ip=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=${instance}" \
  --output text --query 'Reservations[].Instances[].PrivateIpAddress')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance_hostname},${external_ip},${internal_ip} \
  -profile=kubernetes \
  worker-${i}-csr.json | cfssljson -bare worker-${i}
done
```

실행결과:

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

### Controller Manager Client 인증서

`kube-controller-manager` 클라이언트용 인증서와 비공개 키를 생성합니다.

```bash
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "L": "Seoul",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Seoul"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

실행결과:

```
kube-controller-manager-key.pem
kube-controller-manager.pem
```


### Kube Proxy 클라이언트 인증서

`kube-proxy` 클라이언트용 인증서와 비공개 키를 생성합니다.

```bash
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "L": "Seoul",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Seoul"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

결과물:

```
kube-proxy-key.pem
kube-proxy.pem
```

### Scheduler 클라이언트 인증서

`kube-scheduler` 클라이언트용 인증서와 비공개 키를 생성합니다.

```bash
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "L": "Seoul",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Seoul"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

```

실행결과:

```
kube-scheduler-key.pem
kube-scheduler.pem
```


### Kubernetes API Server 인증서

Kubernetes API 용 인증서에는 `kubernetes-the-hard-way` 라는 이름의 고정 IP 주소가 포함됩니다.
이를 통하여 원격 클라이언트들의 인증서를 검증할 수 있습니다.

Kubernetes API Server 용 인증서와 비공개 키를 생성합니다.

```bash
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "L": "Seoul",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Seoul"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

실행결과:

```
kubernetes-key.pem
kubernetes.pem
```

## 서비스 계정 Key Pair

쿠버네티스 컨트롤 매니저는 한쌍의 키를 이용하여 [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) 라는 쿠버네티스 매뉴얼의 내용처럼 서비스 계정 토큰을 생성하고 서명합니다. 

`service-account` 인증서와 비공개 키를 생성합니다.

```bash
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "L": "Seoul",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Seoul"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

```

실행결과:

```
service-account-key.pem
service-account.pem
```


## 클라이언트 및 서버 인증서 배포

생성했던 인증서와 비공개키들을 각 worker 인스턴스안에 복사하여 넣어줍니다.  

```bash
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/
done
```

마찬가지로 master 인스턴스 안에도 생성했던 인증서와 비공개키들을 복사하여 넣어줍니다.

```bash
for instance in master-0 master-1 master-2; do
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa \
    ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ubuntu@${external_ip}:~/
done
```

> 다음 단계에서는 `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, 그리고 `kubelet` 클라이언트 인증서들을 이용하여 클라이언트 인증 설정파일을 생성합니다.

다음: [Kubernetes 인증 처리를 위한 설정 파일 생성](05-kubernetes-configuration-files.md)
