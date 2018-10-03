# etcd 클러스터 준비

Kubernet의 구성 요소들은 상태 정보를 직접 저장하지 않고 [etcd](https://github.com/coreos/etcd) 에 클러스터 저장합니다. 이번에는 3개 노드 등을 부팅하여 고가용성 및 안전한 원격 액세스를 위해 클러스터를 구성합니다.

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

## etcd Cluster Member 준비

### etcd 다운로드

etcd 공식 바이너리 파일을 Github 프로젝트인 [coreos/etcd](https://github.com/coreos/etcd)에서 다운로드합니다.

```bash
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```

`etcd` 서버와 `etcdctl` command line 도구를 압축해제하여 설치합니다.

```bash
tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
```

### etcd Server 설정

```bash
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

인스턴스 내부 IP 주소는 클라이언트 요청을 처리하고 etcd 클러스터 피어와 통신하는 데 사용됩니다. 아래의 명령을 통해 인스턴스의 내부 IP 주소를 검색합니다.

```bash
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
```

각 etcd 멤버들은 etcd 클러스터 안에서 고유 한 이름을 가져야합니다. 현재 인스턴스의 호스트 이름과 일치하도록 etcd 이름을 설정합니다.

```bash
ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${ETCD_NAME}"
```

`etcd.service` 라는 이름으로 systemd 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### etcd 서버 실행

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

> 반드시 위의 명령들은 각 Master 노드 `master-0`, `master-1`, `master-2` 에서 실행해야합니다. 

## etcd 클러스터 검증

etcd 클러스터가 잘 구성되었는지 아래의 명령을 통해 etcd 클러스터 멤버들을 조회합니다.

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> 출력값(예시)

```
3a57933972cb5131, started, master-2, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, master-0, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, master-1, https://10.240.0.11:2380, https://10.240.0.11:2379
```

다음: [Kubernetes 마스터 노드 인스턴스 준비](08-bootstrapping-kubernetes-controllers.md)
