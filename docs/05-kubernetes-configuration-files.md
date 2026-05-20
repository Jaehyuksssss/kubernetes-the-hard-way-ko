# Kubernetes 인증 설정 파일 생성하기

이번 단계에서는 Kubernetes 클라이언트 설정 파일을 생성합니다. 이 파일을 보통 `kubeconfig`라고 부릅니다. 명령은 모두 `jumpbox`의 저장소 디렉터리에서 실행합니다.

```bash
cd /root/kubernetes-the-hard-way-ko
```

4번 단계에서 인증서와 private key를 만들었습니다. 5번 단계에서는 그 인증서를 사용해서 각 Kubernetes 컴포넌트가 API 서버에 접속할 때 필요한 설정 파일을 만듭니다.

쉽게 말하면:

- 4번: “내가 누구인지 증명할 신분증”을 만들었다.
- 5번: “그 신분증을 들고 어느 API 서버에 어떤 이름으로 접속할지”를 적은 설정 파일을 만든다.

## kubeconfig가 하는 일

`kubeconfig`는 Kubernetes 클라이언트가 API 서버에 접속할 때 필요한 정보를 담습니다.

주요 구성은 세 가지입니다.

- `cluster`: 어느 Kubernetes API 서버에 접속할지
- `user`: 어떤 인증서와 key로 인증할지
- `context`: 어떤 cluster와 user 조합을 사용할지

이번 단계에서는 다음 kubeconfig 파일들을 만듭니다.

- `node-0.kubeconfig`
- `node-1.kubeconfig`
- `kube-proxy.kubeconfig`
- `kube-controller-manager.kubeconfig`
- `kube-scheduler.kubeconfig`
- `admin.kubeconfig`

## kubelet kubeconfig 생성

먼저 `node-0`, `node-1`의 kubelet이 사용할 kubeconfig를 생성합니다.

kubelet은 각 worker node에서 실행되며, API 서버에 자기 자신을 node로 등록하고 Pod 상태를 보고합니다. 이때 kubelet의 client certificate는 node 이름과 맞아야 합니다. 그래서 `node-0`은 `node-0.crt`, `node-1`은 `node-1.crt`를 사용합니다.

이 이름이 중요한 이유는 Kubernetes의 Node Authorizer가 `system:node:<node-name>` 형식의 사용자 이름을 기준으로 해당 kubelet이 자기 노드에 대한 권한만 갖도록 제한하기 때문입니다.

```bash
for host in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
    --client-certificate=${host}.crt \
    --client-key=${host}.key \
    --embed-certs=true \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done
```

생성 결과는 다음 파일입니다.

```text
node-0.kubeconfig
node-1.kubeconfig
```

여기서 `--server=https://server.kubernetes.local:6443`는 kubelet이 API 서버에 접속할 주소입니다. `6443`은 Kubernetes API 서버가 HTTPS로 요청을 받는 기본 포트입니다.

`--embed-certs=true`를 사용하는 이유는 인증서 내용을 kubeconfig 파일 안에 직접 포함하기 위해서입니다. 이렇게 하면 별도 인증서 파일 경로에 덜 의존하게 되어 배포가 단순해집니다.

## kube-proxy kubeconfig 생성

`kube-proxy`는 worker node에서 Service 네트워크 규칙을 구성하는 컴포넌트입니다. API 서버에서 Service와 Endpoint 정보를 읽어와서 노드의 네트워크 규칙에 반영합니다.

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-proxy.kubeconfig
}
```

생성 결과는 다음 파일입니다.

```text
kube-proxy.kubeconfig
```

## kube-controller-manager kubeconfig 생성

`kube-controller-manager`는 Deployment, Node, Endpoint, ServiceAccount 등 여러 Kubernetes 리소스의 상태를 원하는 상태로 맞추는 컨트롤러들을 실행합니다. 이 컴포넌트도 API 서버와 계속 통신해야 하므로 kubeconfig가 필요합니다.

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
}
```

생성 결과는 다음 파일입니다.

```text
kube-controller-manager.kubeconfig
```

## kube-scheduler kubeconfig 생성

`kube-scheduler`는 아직 노드가 정해지지 않은 Pod를 보고, 어떤 worker node에 배치할지 결정합니다. 이 과정에서도 API 서버와 통신해야 하므로 kubeconfig가 필요합니다.

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-scheduler.kubeconfig
}
```

생성 결과는 다음 파일입니다.

```text
kube-scheduler.kubeconfig
```

## admin kubeconfig 생성

`admin.kubeconfig`는 관리자가 `kubectl`로 클러스터에 접근할 때 사용할 설정 파일입니다.

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default \
    --kubeconfig=admin.kubeconfig
}
```

생성 결과는 다음 파일입니다.

```text
admin.kubeconfig
```

`admin.kubeconfig`에서 API 서버 주소를 `https://127.0.0.1:6443`으로 설정하는 이유는 이 파일이 나중에 `server` 머신에서 로컬 API 서버에 접근할 때 사용되기 때문입니다.

## 생성 결과 확인

생성된 kubeconfig 파일을 확인합니다.

```bash
ls -1 *.kubeconfig
```

예상 결과는 다음과 같습니다.

```text
admin.kubeconfig
kube-controller-manager.kubeconfig
kube-proxy.kubeconfig
kube-scheduler.kubeconfig
node-0.kubeconfig
node-1.kubeconfig
```

## Kubernetes 설정 파일 배포

이제 생성한 kubeconfig 파일을 각 컴포넌트가 실행될 머신에 복사합니다.

### worker node에 kubelet, kube-proxy kubeconfig 배포

`node-0`, `node-1`에는 `kubelet`과 `kube-proxy`가 실행됩니다. 따라서 각 worker node에 다음 파일을 복사합니다.

- `/var/lib/kubelet/kubeconfig`
- `/var/lib/kube-proxy/kubeconfig`

```bash
for host in node-0 node-1; do
  ssh root@${host} "mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig \
    root@${host}:/var/lib/kube-proxy/kubeconfig

  scp ${host}.kubeconfig \
    root@${host}:/var/lib/kubelet/kubeconfig
done
```

`kube-proxy.kubeconfig`는 모든 worker node에서 같은 파일을 사용합니다. 반면 kubelet kubeconfig는 node 이름과 인증서가 다르므로 `node-0.kubeconfig`, `node-1.kubeconfig`를 각자 맞는 노드에 배포합니다.

### server에 control plane kubeconfig 배포

`server`에는 control plane 컴포넌트가 실행됩니다. 따라서 `admin`, `kube-controller-manager`, `kube-scheduler` kubeconfig를 `server`에 복사합니다.

```bash
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  root@server:~/
```

`kube-api-server`는 별도의 kubeconfig를 만들지 않습니다. API 서버는 클라이언트가 아니라 서버 역할을 하며, 앞 단계에서 배포한 `kube-api-server.crt`, `kube-api-server.key`, `ca.crt` 등을 직접 사용합니다.

## 확인

worker node에 kubeconfig가 들어갔는지 확인합니다.

```bash
for host in node-0 node-1; do
  ssh root@${host} "ls -l /var/lib/kubelet/kubeconfig /var/lib/kube-proxy/kubeconfig"
done
```

server에 kubeconfig가 들어갔는지 확인합니다.

```bash
ssh root@server "ls -l ~/admin.kubeconfig ~/kube-controller-manager.kubeconfig ~/kube-scheduler.kubeconfig"
```

여기까지 완료하면 각 Kubernetes 컴포넌트가 API 서버에 접속할 때 사용할 인증 설정 파일이 준비됩니다.

## 내 실습 기록

이 섹션은 실습을 완료한 뒤 실제 결과를 정리하는 곳입니다.

다음 단계: [데이터 암호화 설정과 키 생성하기](06-data-encryption-keys.md)
