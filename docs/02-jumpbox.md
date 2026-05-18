# jumpbox 설정하기

이번 실습에서는 4대의 머신 중 하나인 `jumpbox`를 설정합니다. `jumpbox`는 앞으로 Kubernetes 클러스터를 만들 때 명령을 실행하는 작업용 호스트입니다.

`jumpbox`를 따로 두는 이유는 작업 위치를 하나로 고정하기 위해서입니다. 인증서 생성, 설정 파일 작성, 바이너리 다운로드, 다른 노드로 파일 복사 같은 작업을 모두 한 곳에서 수행하면 실습 흐름을 따라가기가 쉽습니다. 개인 노트북에서도 비슷한 작업을 할 수 있지만, 실습에서는 환경 차이를 줄이기 위해 `jumpbox`를 기준점으로 사용합니다.

이 문서는 원문처럼 `root` 사용자 기준으로 진행합니다. 이유는 권한 문제를 줄이고 명령어 수를 줄이기 위해서입니다. 실제 운영 환경에서는 보안상 권장되지 않지만, 이 실습에서는 Kubernetes 내부 구조를 배우는 것이 목적이므로 편의성을 우선합니다.

## jumpbox 접속

Multipass로 만든 VM에서는 보통 `ssh root@jumpbox` 대신 `multipass shell`로 접속합니다.

```bash
multipass shell jumpbox
```

접속한 뒤 `root` 셸로 전환합니다.

```bash
sudo -i
```

이렇게 하면 이후 명령을 실행할 때 매번 `sudo`를 붙이지 않아도 됩니다.

현재 위치와 사용자 정보를 확인하고 싶다면 다음 명령을 사용할 수 있습니다.

```bash
whoami
pwd
```

`whoami` 결과가 `root`이면 이후 명령을 그대로 따라가면 됩니다.

## 명령줄 도구 설치

먼저 실습에 필요한 기본 도구를 설치합니다.

```bash
{
  apt-get update
  apt-get -y install wget curl vim openssl git
}
```

이 도구들이 필요한 이유는 다음과 같습니다.

- `wget`, `curl`: Kubernetes 관련 바이너리와 파일을 다운로드할 때 사용합니다.
- `vim`: 설정 파일을 직접 확인하거나 수정할 때 사용합니다.
- `openssl`: 인증서와 키를 다루는 과정에서 사용합니다.
- `git`: 튜토리얼 저장소를 가져올 때 사용합니다.

`apt-get update`를 먼저 실행하는 이유는 패키지 목록을 최신 상태로 갱신하기 위해서입니다. 패키지 목록이 오래되어 있으면 설치 가능한 버전을 찾지 못하거나 설치가 실패할 수 있습니다.

## GitHub 저장소 동기화

이제 Kubernetes The Hard Way 저장소를 `jumpbox` 안으로 가져옵니다. 이 저장소에는 이후 단계에서 사용할 설정 파일, systemd unit 파일, 다운로드 목록이 들어 있습니다.

```bash
git clone --depth 1 \
  https://github.com/jaehyuksssss/kubernetes-the-hard-way-ko.git
```

`--depth 1`을 사용하는 이유는 전체 Git 히스토리를 모두 받지 않고 최신 파일만 받기 위해서입니다. 실습에는 과거 커밋 기록이 필요하지 않으므로 다운로드 시간이 줄어듭니다.

저장소 디렉터리로 이동합니다.

```bash
cd kubernetes-the-hard-way
```

앞으로 대부분의 명령은 이 디렉터리에서 실행합니다. 중간에 위치가 헷갈리면 `pwd`로 현재 경로를 확인합니다.

```bash
pwd
```

예상 출력은 다음과 같습니다.

```text
/root/kubernetes-the-hard-way
```

이 위치를 확인하는 이유는 이후 명령들이 `configs`, `units`, `downloads-*.txt` 같은 상대 경로를 사용하기 때문입니다. 다른 디렉터리에서 실행하면 파일을 찾지 못할 수 있습니다.

## 바이너리 다운로드

이 단계에서는 Kubernetes를 구성하는 여러 컴포넌트의 실행 파일을 다운로드합니다. 다운로드한 파일은 `jumpbox`의 `downloads` 디렉터리에 모아둡니다.

`jumpbox`에 한 번만 다운로드해두는 이유는 같은 파일을 `server`, `node-0`, `node-1`에서 반복해서 다운로드하지 않기 위해서입니다. 이후 단계에서 필요한 바이너리를 각 노드로 복사하면 네트워크 사용량과 실수 가능성을 줄일 수 있습니다.

먼저 현재 머신의 CPU 아키텍처를 기준으로 어떤 파일을 받을지 확인합니다.

```bash
cat downloads-$(dpkg --print-architecture).txt
```

`dpkg --print-architecture`를 사용하는 이유는 머신이 `amd64`인지 `arm64`인지에 따라 받아야 하는 바이너리가 다르기 때문입니다. Apple Silicon Mac에서 Multipass를 사용하는 경우 `arm64`가 나올 수 있고, Intel/AMD 기반 환경에서는 보통 `amd64`가 나옵니다.

바이너리를 `downloads` 디렉터리에 다운로드합니다.

```bash
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-$(dpkg --print-architecture).txt
```

옵션의 의미는 다음과 같습니다.

- `--https-only`: HTTPS 주소만 사용합니다.
- `--timestamping`: 이미 받은 파일이 있으면 불필요하게 다시 받지 않습니다.
- `-P downloads`: 파일을 `downloads` 디렉터리에 저장합니다.
- `-i ...`: 다운로드할 URL 목록을 파일에서 읽습니다.

다운로드 용량이 500MB 이상일 수 있으므로 인터넷 속도에 따라 시간이 걸릴 수 있습니다. 다운로드가 끝나면 파일 목록을 확인합니다.

```bash
ls -oh downloads
```

## 바이너리 압축 해제와 정리

다운로드한 압축 파일에서 실제 실행 파일을 꺼내고, 역할별 디렉터리로 정리합니다.

```bash
{
  ARCH=$(dpkg --print-architecture)
  mkdir -p downloads/{client,cni-plugins,controller,worker}
  tar -xvf downloads/crictl-v1.32.0-linux-${ARCH}.tar.gz \
    -C downloads/worker/
  tar -xvf downloads/containerd-2.1.0-beta.0-linux-${ARCH}.tar.gz \
    --strip-components 1 \
    -C downloads/worker/
  tar -xvf downloads/cni-plugins-linux-${ARCH}-v1.6.2.tgz \
    -C downloads/cni-plugins/
  tar -xvf downloads/etcd-v3.6.0-rc.3-linux-${ARCH}.tar.gz \
    -C downloads/ \
    --strip-components 1 \
    etcd-v3.6.0-rc.3-linux-${ARCH}/etcdctl \
    etcd-v3.6.0-rc.3-linux-${ARCH}/etcd
  mv downloads/{etcdctl,kubectl} downloads/client/
  mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \
    downloads/controller/
  mv downloads/{kubelet,kube-proxy} downloads/worker/
  mv downloads/runc.${ARCH} downloads/worker/runc
}
```

디렉터리를 역할별로 나누는 이유는 각 바이너리가 설치될 위치가 다르기 때문입니다.

- `client`: `kubectl`, `etcdctl`처럼 관리자가 사용하는 클라이언트 도구
- `controller`: `server`에 설치될 control plane 컴포넌트
- `worker`: worker node에 설치될 `kubelet`, `kube-proxy`, `containerd`, `runc`
- `cni-plugins`: Pod 네트워크 구성을 위한 CNI 플러그인

압축 해제가 끝났다면 원본 압축 파일은 더 이상 필요하지 않으므로 삭제합니다.

```bash
rm -rf downloads/*gz
```

삭제하는 이유는 디스크 공간을 아끼고, 이후 파일 목록을 볼 때 실제 사용할 실행 파일만 남기기 위해서입니다.

이제 실행 파일에 실행 권한을 부여합니다.

```bash
{
  chmod +x downloads/{client,cni-plugins,controller,worker}/*
}
```

Linux에서는 파일이 실행 파일 형태로 존재하더라도 실행 권한이 없으면 명령으로 실행할 수 없습니다. `chmod +x`는 이후 각 바이너리를 실행할 수 있게 만드는 단계입니다.

## kubectl 설치

`kubectl`은 Kubernetes API 서버와 통신하는 공식 CLI 도구입니다. 클러스터가 만들어진 뒤 상태 확인, Pod 생성, smoke test 등에 사용합니다.

`kubectl` 바이너리를 시스템 PATH에 포함된 `/usr/local/bin`으로 복사합니다.

```bash
{
  cp downloads/client/kubectl /usr/local/bin/
}
```

`/usr/local/bin`에 복사하는 이유는 어느 디렉터리에 있든 `kubectl` 명령을 바로 실행할 수 있게 하기 위해서입니다.

설치가 잘 되었는지 확인합니다.

```bash
kubectl version --client
```

예상 출력은 다음과 비슷합니다.

```text
Client Version: v1.32.3
Kustomize Version: v5.5.0
```

여기까지 완료하면 `jumpbox`에는 이후 실습을 진행하는 데 필요한 기본 도구와 Kubernetes 바이너리가 준비된 상태입니다.

다음 단계: [컴퓨트 리소스 준비하기](03-compute-resources.md)
