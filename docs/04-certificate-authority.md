# CA 준비와 TLS 인증서 생성하기

이번 단계에서는 `openssl`을 사용해 Kubernetes 클러스터에서 사용할 CA와 TLS 인증서를 생성합니다. 명령은 모두 `jumpbox`의 저장소 디렉터리에서 실행합니다.

```bash
cd /root/kubernetes-the-hard-way-ko
```

Kubernetes는 여러 컴포넌트가 네트워크로 서로 통신하는 시스템입니다. `kube-apiserver`, `kubelet`, `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, `etcd` 같은 구성요소들이 서로 요청을 주고받습니다. 이때 아무나 클러스터 내부 API에 접근하면 안 되기 때문에 TLS 인증서를 사용해 서로의 신원을 확인합니다.

이번 단계의 목적은 크게 두 가지입니다.

- 클러스터 내부에서 신뢰 기준이 될 CA를 만든다.
- 각 Kubernetes 컴포넌트와 사용자에게 사용할 인증서와 private key를 만든다.

## CA가 필요한 이유

CA는 Certificate Authority의 약자입니다. 쉽게 말하면 “이 인증서는 내가 신뢰해도 된다고 보증하는 주체”입니다.

Kubernetes 컴포넌트들은 서로 통신할 때 상대방의 인증서를 확인합니다. 이때 인증서가 우리가 만든 CA에 의해 서명되어 있으면, 같은 클러스터에 속한 신뢰 가능한 컴포넌트라고 판단할 수 있습니다.

이번 실습에서는 자체 서명된 self-signed CA를 만듭니다. 실습용으로는 충분하지만, 실제 운영 환경에서는 보안 정책에 맞는 CA 관리 방식이 필요합니다.

## ca.conf 확인

이 저장소에는 인증서 생성을 쉽게 하기 위한 `ca.conf` 파일이 포함되어 있습니다. 이 파일에는 CA와 각 Kubernetes 컴포넌트 인증서에 들어갈 subject, 조직 정보, DNS 이름, IP 주소 같은 설정이 들어 있습니다.

먼저 내용을 확인합니다.

```bash
cat ca.conf
```

처음부터 `ca.conf`의 모든 내용을 이해할 필요는 없습니다. 다만 이 파일이 “어떤 인증서를 어떤 이름과 권한으로 만들지”를 정의한다는 점은 기억해두면 좋습니다.

특히 `kube-api-server`, `node-0`, `node-1`, `admin` 같은 section은 이후 인증서 생성 명령에서 사용됩니다.

## CA 인증서와 private key 생성

모든 인증서의 출발점이 되는 CA private key와 CA 인증서를 생성합니다.

```bash
{
  openssl genrsa -out ca.key 4096
  openssl req -x509 -new -sha512 -noenc \
    -key ca.key -days 3653 \
    -config ca.conf \
    -out ca.crt
}
```

각 파일의 의미는 다음과 같습니다.

- `ca.key`: CA의 private key입니다. 다른 인증서를 서명할 때 사용합니다.
- `ca.crt`: CA의 root certificate입니다. 각 컴포넌트가 “어떤 CA를 신뢰할지” 판단할 때 사용합니다.

`ca.key`는 매우 민감한 파일입니다. 이 키를 가진 사람은 클러스터에서 신뢰되는 인증서를 새로 만들 수 있기 때문입니다. 실습에서는 편의를 위해 `jumpbox`에 보관하지만, 실제 운영 환경에서는 엄격하게 보호해야 합니다.

생성 결과는 다음 파일입니다.

```text
ca.crt
ca.key
```

## 클라이언트와 서버 인증서 생성

이제 각 Kubernetes 컴포넌트와 `admin` 사용자용 인증서를 생성합니다.

```bash
certs=(
  "admin" "node-0" "node-1"
  "kube-proxy" "kube-scheduler"
  "kube-controller-manager"
  "kube-api-server"
  "service-accounts"
)
```

각 인증서의 용도는 다음과 같습니다.

- `admin`: 관리자 사용자가 `kubectl`로 Kubernetes API에 접근할 때 사용합니다.
- `node-0`, `node-1`: 각 worker node의 `kubelet`이 자신의 신원을 증명할 때 사용합니다.
- `kube-proxy`: worker node에서 서비스 트래픽 규칙을 구성하는 `kube-proxy`가 API 서버에 접근할 때 사용합니다.
- `kube-scheduler`: scheduler가 API 서버와 통신할 때 사용합니다.
- `kube-controller-manager`: controller manager가 API 서버와 통신할 때 사용합니다.
- `kube-api-server`: Kubernetes API 서버가 HTTPS 서버로 동작할 때 사용합니다.
- `service-accounts`: ServiceAccount token 서명에 사용합니다.

다음 반복문은 각 항목마다 private key, CSR, CA가 서명한 인증서를 생성합니다.

```bash
for i in ${certs[*]}; do
  openssl genrsa -out "${i}.key" 4096

  openssl req -new -key "${i}.key" -sha256 \
    -config "ca.conf" -section ${i} \
    -out "${i}.csr"

  openssl x509 -req -days 3653 -in "${i}.csr" \
    -copy_extensions copyall \
    -sha256 -CA "ca.crt" \
    -CAkey "ca.key" \
    -CAcreateserial \
    -out "${i}.crt"
done
```

흐름을 풀면 다음과 같습니다.

1. `${i}.key` private key를 만든다.
2. `${i}.csr` certificate signing request를 만든다.
3. CA의 `ca.key`, `ca.crt`로 CSR을 서명해 `${i}.crt` 인증서를 만든다.

생성된 파일을 확인합니다.

```bash
ls -1 *.crt *.key *.csr
```

여기서 `.key` 파일은 private key이고, `.crt` 파일은 인증서입니다. `.csr` 파일은 인증서 요청 파일이며, 인증서가 만들어진 뒤에는 실습 진행에 직접 사용되지는 않습니다.

## 인증서 배포

이제 생성한 인증서와 private key를 각 머신의 적절한 위치에 복사합니다.

인증서를 배포하는 이유는 각 Kubernetes 컴포넌트가 실행될 머신에서 자신의 인증서와 CA 인증서를 읽을 수 있어야 하기 때문입니다. 예를 들어 `node-0`의 `kubelet`은 `node-0`용 인증서와 key를 가지고 API 서버에 자신을 증명합니다.

### worker node에 kubelet 인증서 배포

`node-0`, `node-1`에는 kubelet이 사용할 인증서를 복사합니다.

```bash
for host in node-0 node-1; do
  ssh root@${host} mkdir /var/lib/kubelet/

  scp ca.crt root@${host}:/var/lib/kubelet/

  scp ${host}.crt \
    root@${host}:/var/lib/kubelet/kubelet.crt

  scp ${host}.key \
    root@${host}:/var/lib/kubelet/kubelet.key
done
```

복사 결과는 각 worker node에서 다음과 같은 형태가 됩니다.

```text
/var/lib/kubelet/ca.crt
/var/lib/kubelet/kubelet.crt
/var/lib/kubelet/kubelet.key
```

`node-0.crt`와 `node-1.crt`를 각각 `kubelet.crt`라는 이름으로 복사하는 이유는, 각 노드 안에서는 kubelet이 일정한 경로와 파일명으로 인증서를 읽도록 맞추기 위해서입니다.

### server에 API 서버와 service account 인증서 배포

`server`에는 API 서버와 service account에 필요한 인증서를 복사합니다.

```bash
scp \
  ca.key ca.crt \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  root@server:~/
```

`server`에 복사되는 파일의 의미는 다음과 같습니다.

- `ca.crt`: API 서버가 클라이언트 인증서를 검증할 때 사용합니다.
- `ca.key`: 이후 일부 인증서나 설정에서 CA 서명이 필요할 때 사용됩니다.
- `kube-api-server.key`, `kube-api-server.crt`: API 서버가 HTTPS 서버로 동작할 때 사용합니다.
- `service-accounts.key`, `service-accounts.crt`: ServiceAccount token 서명과 검증에 사용합니다.

> `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, `kubelet`의 client 인증서는 다음 단계에서 kubeconfig 파일을 만들 때 사용합니다.

## 확인

worker node에 파일이 들어갔는지 확인합니다.

```bash
for host in node-0 node-1; do
  ssh root@${host} "ls -l /var/lib/kubelet"
done
```

`server`에 파일이 들어갔는지 확인합니다.

```bash
ssh root@server "ls -l ~/ca.* ~/kube-api-server.* ~/service-accounts.*"
```

여기까지 완료하면 Kubernetes 컴포넌트들이 서로 인증할 때 사용할 기본 TLS 재료가 준비된 상태입니다.

## 내 실습 기록

이 섹션은 실습을 완료한 뒤 실제 결과를 정리하는 곳입니다.

다음 단계: [Kubernetes 인증 설정 파일 생성하기](05-kubernetes-configuration-files.md)
