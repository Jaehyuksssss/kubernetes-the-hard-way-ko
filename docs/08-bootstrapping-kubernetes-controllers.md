# Kubernetes Control Plane 부트스트랩하기

이번 단계에서는 `server` 머신에 Kubernetes control plane을 구성합니다. 설치되는 주요 컴포넌트는 다음과 같습니다.

- `kube-apiserver`
- `kube-controller-manager`
- `kube-scheduler`

control plane은 Kubernetes 클러스터의 두뇌에 해당합니다. 사용자의 요청을 받고, 클러스터 상태를 저장하고, Pod를 어떤 노드에 배치할지 결정하고, 실제 상태가 원하는 상태와 같아지도록 계속 조정합니다.

## Control Plane 컴포넌트 역할

`kube-apiserver`는 Kubernetes API의 입구입니다. `kubectl`, kubelet, scheduler, controller manager 등 거의 모든 컴포넌트는 API 서버와 통신합니다. API 서버는 요청을 검증하고, 필요한 상태를 etcd에 저장하거나 읽습니다.

`kube-controller-manager`는 여러 컨트롤러를 실행합니다. 예를 들어 Deployment, Node, ServiceAccount 같은 리소스가 원하는 상태를 유지하도록 계속 감시하고 조정합니다.

`kube-scheduler`는 아직 노드가 정해지지 않은 Pod를 보고, 어떤 worker node에 배치할지 결정합니다.

7번에서 etcd를 먼저 실행한 이유는 API 서버가 클러스터 상태를 저장할 곳이 필요하기 때문입니다. 이제 etcd가 준비되었으므로 control plane을 시작할 수 있습니다.

## 사전 준비

먼저 `jumpbox`에서 Kubernetes control plane 바이너리와 systemd unit 파일, 설정 파일을 `server`로 복사합니다.

이 명령은 `jumpbox`에서 실행합니다.

```bash
scp \
  downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  units/kube-apiserver.service \
  units/kube-controller-manager.service \
  units/kube-scheduler.service \
  configs/kube-scheduler.yaml \
  configs/kube-apiserver-to-kubelet.yaml \
  root@server:~/
```

각 파일의 의미는 다음과 같습니다.

- `kube-apiserver`: Kubernetes API 서버 실행 파일
- `kube-controller-manager`: controller manager 실행 파일
- `kube-scheduler`: scheduler 실행 파일
- `kubectl`: Kubernetes API와 통신하는 CLI 도구
- `kube-apiserver.service`: API 서버 systemd unit
- `kube-controller-manager.service`: controller manager systemd unit
- `kube-scheduler.service`: scheduler systemd unit
- `kube-scheduler.yaml`: scheduler 설정 파일
- `kube-apiserver-to-kubelet.yaml`: API 서버가 kubelet API에 접근할 수 있도록 하는 RBAC 설정

복사가 끝나면 `server`에 접속합니다.

```bash
ssh root@server
```

이후 control plane 설치 명령은 반드시 `server`에서 실행합니다. 프롬프트가 다음과 같은지 확인합니다.

```text
root@server:~#
```

## Kubernetes 설정 디렉터리 생성

Kubernetes 설정 파일을 둘 디렉터리를 만듭니다.

```bash
mkdir -p /etc/kubernetes/config
```

`/etc/kubernetes/config`는 scheduler 같은 컴포넌트 설정 파일을 둘 위치입니다.

## Kubernetes control plane 바이너리 설치

`server`에 복사된 실행 파일들을 `/usr/local/bin`으로 옮깁니다.

```bash
{
  mv kube-apiserver \
    kube-controller-manager \
    kube-scheduler kubectl \
    /usr/local/bin/
}
```

`/usr/local/bin`으로 옮기는 이유는 systemd unit 파일과 셸에서 실행 파일을 안정적인 경로로 찾을 수 있게 하기 위해서입니다.

설치 확인:

```bash
which kube-apiserver
which kube-controller-manager
which kube-scheduler
which kubectl
```

## Kubernetes API Server 설정

API 서버가 사용할 인증서, key, encryption config를 `/var/lib/kubernetes`로 옮깁니다.

```bash
{
  mkdir -p /var/lib/kubernetes/

  mv ca.crt ca.key \
    kube-api-server.key kube-api-server.crt \
    service-accounts.key service-accounts.crt \
    encryption-config.yaml \
    /var/lib/kubernetes/
}
```

각 파일의 역할은 다음과 같습니다.

- `ca.crt`, `ca.key`: 클러스터 CA 인증서와 key
- `kube-api-server.key`, `kube-api-server.crt`: API 서버가 HTTPS 서버로 동작할 때 사용하는 인증서와 key
- `service-accounts.key`, `service-accounts.crt`: ServiceAccount token 서명과 검증에 사용
- `encryption-config.yaml`: Secret 데이터를 etcd에 암호화해서 저장하기 위한 설정

API 서버 systemd unit 파일을 systemd 위치로 옮깁니다.

```bash
mv kube-apiserver.service \
  /etc/systemd/system/kube-apiserver.service
```

## Kubernetes Controller Manager 설정

`kube-controller-manager`가 API 서버에 접속할 때 사용할 kubeconfig를 `/var/lib/kubernetes`로 옮깁니다.

```bash
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

systemd unit 파일을 옮깁니다.

```bash
mv kube-controller-manager.service /etc/systemd/system/
```

## Kubernetes Scheduler 설정

`kube-scheduler`가 API 서버에 접속할 때 사용할 kubeconfig를 `/var/lib/kubernetes`로 옮깁니다.

```bash
mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

scheduler 설정 파일을 `/etc/kubernetes/config`로 옮깁니다.

```bash
mv kube-scheduler.yaml /etc/kubernetes/config/
```

systemd unit 파일을 옮깁니다.

```bash
mv kube-scheduler.service /etc/systemd/system/
```

## Control Plane 서비스 시작

systemd 설정을 다시 읽고, control plane 서비스들을 활성화한 뒤 시작합니다.

```bash
{
  systemctl daemon-reload

  systemctl enable kube-apiserver \
    kube-controller-manager kube-scheduler

  systemctl start kube-apiserver \
    kube-controller-manager kube-scheduler
}
```

API 서버가 완전히 초기화되는 데 몇 초 정도 걸릴 수 있습니다. 최대 10초 정도 기다린 뒤 확인합니다.

```bash
systemctl is-active kube-apiserver
systemctl is-active kube-controller-manager
systemctl is-active kube-scheduler
```

자세한 상태를 보려면 다음 명령을 사용할 수 있습니다.

```bash
systemctl status kube-apiserver --no-pager
systemctl status kube-controller-manager --no-pager
systemctl status kube-scheduler --no-pager
```

문제가 있을 때 로그를 확인하려면 `journalctl`을 사용합니다.

```bash
journalctl -u kube-apiserver --no-pager
journalctl -u kube-controller-manager --no-pager
journalctl -u kube-scheduler --no-pager
```

## Control Plane 동작 확인

`server`에서 `kubectl`로 control plane 정보를 확인합니다.

```bash
kubectl cluster-info \
  --kubeconfig admin.kubeconfig
```

예상 출력은 다음과 같습니다.

```text
Kubernetes control plane is running at https://127.0.0.1:6443
```

여기서 `admin.kubeconfig`는 5번 단계에서 `server`의 `/root`로 복사해둔 파일입니다. API 서버 주소가 `https://127.0.0.1:6443`으로 되어 있으므로 `server` 안에서 로컬 API 서버에 접속하는 데 사용됩니다.

## Kubelet 접근 권한을 위한 RBAC 설정

이제 API 서버가 worker node의 kubelet API에 접근할 수 있도록 RBAC 권한을 설정합니다.

이 권한이 필요한 이유는 API 서버가 다음과 같은 작업을 할 때 kubelet API에 접근해야 하기 때문입니다.

- Pod 로그 조회
- Pod 안에서 명령 실행
- metrics 조회
- kubelet을 통한 일부 상태 확인

이 실습에서는 kubelet의 `--authorization-mode`가 `Webhook`으로 설정됩니다. Webhook 모드는 Kubernetes API의 `SubjectAccessReview`를 사용해 접근 권한을 판단합니다.

다음 명령은 `server`에서 한 번만 실행하면 됩니다.

```bash
kubectl apply -f kube-apiserver-to-kubelet.yaml \
  --kubeconfig admin.kubeconfig
```

이 명령은 `system:kube-apiserver-to-kubelet` ClusterRole과 관련 binding을 생성합니다.

## 외부에서 API 서버 확인

마지막으로 `jumpbox`에서 API 서버의 version endpoint에 요청을 보내 control plane이 외부에서도 응답하는지 확인합니다.

먼저 `server`에서 나와 `jumpbox`로 돌아갑니다.

```bash
exit
```

프롬프트가 다음과 같은지 확인합니다.

```text
root@jumpbox:~/kubernetes-the-hard-way-ko#
```

그다음 `jumpbox`에서 실행합니다.

```bash
curl --cacert ca.crt \
  https://server.kubernetes.local:6443/version
```

예상 출력은 다음과 비슷합니다.

```text
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  "gitCommit": "32cc146f75aad04beaaa245a7157eb35063a9f99",
  "gitTreeState": "clean",
  "buildDate": "2025-03-11T19:52:21Z",
  "goVersion": "go1.23.6",
  "compiler": "gc",
  "platform": "linux/arm64"
}
```

`--cacert ca.crt`를 넣는 이유는 우리가 직접 만든 CA로 서명한 API 서버 인증서를 curl이 신뢰하도록 하기 위해서입니다.

## 내 실습 기록

이 섹션은 실습을 완료한 뒤 실제 결과를 정리하는 곳입니다.

다음 단계: [Kubernetes Worker Node 부트스트랩하기](09-bootstrapping-kubernetes-workers.md)
