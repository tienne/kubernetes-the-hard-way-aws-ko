# Provisioning Compute Resources

쿠버네티스는 컨테이너들이 실행되여 작업을 하는 worker node 들과 쿠버네티스 호스트들을 컨트롤하기 위한 물리 서버가 필요합니다.
이번 단계에서는 안전하고 고가용성의 쿠버네티스 클러스터를 실행하기 위한 컴퓨터 리소스를 단일 [region](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/using-regions-availability-zones.html) 에 프로비저닝하도록 하겠습니다. 

> 저는 aws-cli의 기본 리전을 ap-northeast-2 로 설정하였습니다.

## Networking

Kubernets [네트워킹 모델](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model)은 컨테이너와 노드가 서로 통신할 수 있는 평면 네트워크를 가정합니다. 이러한 방식이 적합하지 않은 경우 [네트워크 정책](https://kubernetes.io/docs/concepts/services-networking/network-policies/)을 컨테이너 그룹간의 서로 외부 네트워크 엔드포인트와 통신할 수 있는 방법을 제한할 수 있습니다.

> 네트워크 정책 설정은 이 가이드의 범위를 벗어납니다.
 
### VPC

이번 단계에서는 전용 VPC([Virtual Private Cloud](https://cloud.google.com/vpc/docs/vpc#networks)) 네트워크가 Kubernetes 클러스터를 호스팅하도록 설정합니다.

아래의 명령어를 통해 aws에 `kubernetes-the-hard-way` 사용자 정의 VPC 네트워크를 생성합니다.

```sh
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.240.0.0/24 --output text --query 'Vpc.VpcId')
aws ec2 create-tags --resources ${VPC_ID} --tags Key=Name,Value=kubernetes-the-hard-way
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-support '{"Value": true}'
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-hostnames '{"Value": true}'
```

### Subnet
  
서브넷은 Kubernetes 클러스터 노드들의 IP주소로 사용하기 위한 충분한 범위로 프로비저닝해야합니다.

아래의 명령어로 kubernetes-the-hard-way VPC 네트워크에의 서브넷을 생성합니다.

```sh
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.240.0.0/24 \
  --output text --query 'Subnet.SubnetId')
aws ec2 create-tags --resources ${SUBNET_ID} --tags Key=Name,Value=kubernetes
```

### Internet Gateway

생성한 VPC 외부 인터넷 트래픽을 제어할 Internet Gateway를 생성합니다.
 
```sh
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway --output text --query 'InternetGateway.InternetGatewayId')
aws ec2 create-tags --resources ${INTERNET_GATEWAY_ID} --tags Key=Name,Value=kubernetes
aws ec2 attach-internet-gateway --internet-gateway-id ${INTERNET_GATEWAY_ID} --vpc-id ${VPC_ID}
```

### Route Tables

Route table 를 생성하여 생성해둔 Internet Gateway 를 VPC 와 연결합니다.  

```sh
ROUTE_TABLE_ID=$(aws ec2 create-route-table --vpc-id ${VPC_ID} --output text --query 'RouteTable.RouteTableId')
aws ec2 create-tags --resources ${ROUTE_TABLE_ID} --tags Key=Name,Value=kubernetes
aws ec2 associate-route-table --route-table-id ${ROUTE_TABLE_ID} --subnet-id ${SUBNET_ID}
aws ec2 create-route --route-table-id ${ROUTE_TABLE_ID} --destination-cidr-block 0.0.0.0/0 --gateway-id ${INTERNET_GATEWAY_ID}
```

### Security Groups (방화벽 규칙)

아래의 명령어를 통하여 모든 VPC 내부에서 모든 프로토콜을 허용하는 방화벽 규칙을 생성합니다.
 
```sh
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name kubernetes \
  --description "Kubernetes security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')
aws ec2 create-tags --resources ${SECURITY_GROUP_ID} --tags Key=Name,Value=kubernetes
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.240.0.0/24
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.200.0.0/16
```

SSH, ICMP 및 HTTPS를 허용하는 외부 방화벽 규칙을 만듭니다.

```sh
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 6443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol icmp --port -1 --cidr 0.0.0.0/0
```

### Kubernetes Public Access - Create a Network Load Balancer



```sh
LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
  --name kubernetes \
  --subnets ${SUBNET_ID} \
  --scheme internet-facing \
  --type network \
  --output text --query 'LoadBalancers[].LoadBalancerArn')
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name kubernetes \
  --protocol TCP \
  --port 6443 \
  --vpc-id ${VPC_ID} \
  --target-type ip \
  --output text --query 'TargetGroups[].TargetGroupArn')
aws elbv2 register-targets --target-group-arn ${TARGET_GROUP_ARN} --targets Id=10.240.0.1{0,1,2}
aws elbv2 create-listener \
  --load-balancer-arn ${LOAD_BALANCER_ARN} \
  --protocol TCP \
  --port 6443 \
  --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
  --output text --query 'Listeners[].ListenerArn'
```

```sh
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
```

## Compute Instances

### Instance Image

Kubernetes 클러스터들을 띄울때 사용할 AWS ami image ID를 조회합니다. 

저는 canonical 사(ubuntu 유통 회사)에서 제공하는 ubuntu 16.04 버전 AMI 이미지를 사용하겠습니다.

```sh
IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
  --filters \
  'Name=root-device-type,Values=ebs' \
  'Name=architecture,Values=x86_64' \
  'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*' \
  | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
```

### SSH Key Pair

kubernetes 각 클러스터에 ssh 접속을 하기 위한 Key Pair 를 아래의 명령어를 통해 생성합니다.

```sh
aws ec2 create-key-pair --key-name kubernetes --output text --query 'KeyMaterial' > kubernetes.id_rsa
chmod 600 kubernetes.id_rsa
```

### Kubernetes 마스터

`t2.micro` 타입의 인스턴스로 Kubernetes Master 3대를 생성합니다.

```sh
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name kubernetes \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 10.240.0.1${i} \
    --user-data "name=master-${i}" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
  aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=master-${i}"
  echo "master-${i} created "
done
```

### Kubernetes Workers

```sh
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name kubernetes \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 10.240.0.2${i} \
    --user-data "name=worker-${i}|pod-cidr=10.200.${i}.0/24" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
  aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=worker-${i}"
  echo "worker-${i} created"
done
```

다음: [Certificate Authority](04-certificate-authority.md)
