# Client 도구들 설치

이번 단계에선 이 가이드에 필요한 필수 command line 도구들인 [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl),
그리고 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl) 를 설치하도록 하겠습니다.
 
## CFSSL 설치

`cfssl` 과 `cfssljson` 은 [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) 를 프로비저닝하고 TLS 인증서를 생성하기 위해 사용되는 command line 도구입니다.  

[cfssl 저장소](https://pkg.cfssl.org) 에서 `cfssl` 과 `cfssljson` 를 설치해주시기 바랍니다.  

### OS X

```bash
curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_darwin-amd64
curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_darwin-amd64
```

```bash
chmod +x cfssl cfssljson
```

```bash
sudo mv cfssl cfssljson /usr/local/bin/
```

일부 OS X 사용자들은 이미 빌드된 바이너리 설치방식에 문제가 발생될 수 있습니다.
이런 경우 [Homebrew](https://brew.sh) 로 설치하시는걸 권장합니다.

```bash
brew install cfssl
```

### Linux

```bash
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
```

```bash
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
```

```bash
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
```

```bash
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

### 설치 확인

설치된 `cfssl` 의 버전이 1.2.0 이상인지 아래의 명령어를 입력하여 확인해주세요.

```bash
cfssl version
```

> 결과

```
Version: 1.2.0
Revision: dev
Runtime: go1.6
```

> cfssljson 은 현재 설치된 버전을 출력하는 기능을 제공하지 않습니다.

## Install kubectl

`kubectl` command line 도구는 Kubernetes API 서버에 작업을 실행시킬때 사용합니다. 공식 바이너리 파일의 `kubectl` 를 아래를 참고하여 다운로드 해주세요.

### OS X

```bash
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.11.3/bin/darwin/amd64/kubectl
```

```bash
chmod +x kubectl
```

```bash
sudo mv kubectl /usr/local/bin/
```

저는 mac os 에서 [Homebrew](https://brew.sh) 를 이용해서 설치했습니다.

```bash
brew install kubernetes-cli
```

### Linux

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kubectl
```

```bash
chmod +x kubectl
```

```bash
sudo mv kubectl /usr/local/bin/
```

### 설치확인

설치된 `kubectl` 이 1.11.3 버전 이상인지 확인해 주세요.

```bash
kubectl version --client
```

> output

```
Client Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.3", GitCommit:"a4529464e4629c21224b3d52edfe0ea91b072862", GitTreeState:"clean", BuildDate:"2018-09-10T11:44:36Z", GoVersion:"go1.11", Compiler:"gc", Platform:"darwin/amd64"}
```

다음: [서버 준비](03-compute-resources.md)
