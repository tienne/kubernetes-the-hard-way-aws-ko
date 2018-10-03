# 데이터 암호화 설정 및 키 생성

Kubernetes 는 클러스터의 상태, 응용 프로그램 구성 및 비공개 정보등 다양한 데이터를 저장하는데 이 데이터들을 [암호화](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) 하는 기능을 지원합니다.

이러한 암호화 기능을 활용하기 위해 암호화 키와 [설정 파일](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) 을 생성합니다. 

## 암호화 키

암호화 키를 생성합니다.

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## 암호화 설정 파일

암호화 설정 파일인 `encryption-config.yaml` 를 생성합니다.

```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

생성된 암호화 설정 파일 `encryption-config.yaml` 를 각 마스터 노드 인스턴스에 복사합니다.

```bash
for instance in master-0 master-1 master-2; do
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  
  scp -i kubernetes.id_rsa encryption-config.yaml ubuntu@${external_ip}:~/
done
```

다음: [etcd 클러스터 준비](07-bootstrapping-etcd.md)
