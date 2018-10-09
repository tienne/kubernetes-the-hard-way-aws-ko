# 쿠버네티스 Worker 노드 준비

이번에는 쿠버네티스 Worker 노드들을 준비하겠습니다. 각 Worker 노드들에는 [runc](https://github.com/opencontainers/runc), [gVisor](https://github.com/google/gvisor), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), 그리고 [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies) 등이 설치됩니다.

## 준비사항

아래의 명령어 부터는 Worker instance 인 `worker-0`, `worker-1`, 그리고 `worker-2` 에서 실행되어야합니다. 아래의 예시와 같이 `ssh` 접속하기 위한 IP 를 확인합니다.

```bash
for instance in worker-0 worker-1 worker-2; do
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

## Kubernetes Worker 노드 구성 

의존성을 가지고 있는 ubuntu 패키지들을 설치합니다.

```bash
sudo apt-get update
sudo apt-get -y install socat conntrack ipset
```

> socat 바이너리는 `kubectl port-forward` 명령을 지원합니다.

### Worker 바이너리 다운로드 및 설치

```bash
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-incubator/cri-tools/releases/download/v1.11.1/crictl-v1.11.1-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-the-hard-way/runsc \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc5/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.2.0-beta.2/containerd-1.2.0-beta.2.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.11.2/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.11.2/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.11.2/bin/linux/amd64/kubelet
```

설치할 디렉토리를 생성합니다.

```bash
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

그후 worker 바이너리를 설치합니다.

```bash
chmod +x kubectl kube-proxy kubelet runc.amd64 runsc
sudo mv runc.amd64 runc
sudo mv kubectl kube-proxy kubelet runc runsc /usr/local/bin/
sudo tar -xvf crictl-v1.11.1-linux-amd64.tar.gz -C /usr/local/bin/
sudo tar -xvf cni-plugins-amd64-v0.7.1.tgz -C /opt/cni/bin/
sudo tar -xvf containerd-1.2.0-beta.2.linux-amd64.tar.gz -C /
```

### CNI Networking 설정

현재 인스턴스에 대한 포드 CIDR 범위를 조회합니다.

```bash
POD_CIDR=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^pod-cidr" | cut -d"=" -f2)
echo "${POD_CIDR}"
```

`bridge` network 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

`loopback` network 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF
```

### containerd 설정

`containerd` 디렉토리 및 설정 파일을 생성합니다.

```bash
sudo mkdir -p /etc/containerd/
```

```bash
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
EOF
```

> 신뢰할 수 없는 워크로드는 gVisor(runsc) 런타임으로 실행됩니다.

`containerd.service` systemd 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Kubelet 설정

```bash
WORKER_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
| tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${WORKER_NAME}"

sudo mv ${WORKER_NAME}-key.pem ${WORKER_NAME}.pem /var/lib/kubelet/
sudo mv ${WORKER_NAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/
```

`kubelet-config.yaml` 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${WORKER_NAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${WORKER_NAME}-key.pem"
EOF
```

`kubelet.service` systemd 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetes Proxy 설정

```bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

`kube-proxy-config.yaml` 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

`kube-proxy.service` systemd 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Worker 서비스들 실행

```bash
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

> 각 Worker node 인스턴스인 `worker-0`, `worker-1`, 그리고 `worker-2` 에서 실행해야하는걸 잊지마세요.

## 실행 확인

> worker 노드들이 실행되었는지 확인을 위해서는 master node 에서 확인이 가능합니다. 마스터0 인스턴스에 접속하여 아래의 절차를 통해 확인합니다. 

등록된 쿠버네티스 노드 리스트를 확인합니다.

```bash
external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=master-0" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i kubernetes.id_rsa ubuntu@${external_ip}

kubectl get nodes --kubeconfig admin.kubeconfig
```

> 실행결과

```
NAME             STATUS    ROLES     AGE       VERSION
ip-10-240-0-20   Ready     <none>    21s       v1.11.2
ip-10-240-0-21   Ready     <none>    25s       v1.11.2
ip-10-240-0-22   Ready     <none>    25s       v1.11.2
```

다음: [원격 접근을 위한 kubectl 설정](10-configuring-kubectl.md)
