# 사전 준비

이 실습에서는 튜토리얼을 따라가기 위해 필요한 머신 요구사항을 확인합니다.

## 가상 머신 또는 물리 머신

이 튜토리얼은 Debian 12(bookworm)를 실행하는 ARM64 또는 AMD64 기반 가상 머신 또는 물리 머신 4대가 필요합니다. 아래 표는 각 머신의 역할과 CPU, 메모리, 스토리지 요구사항입니다.

머신을 4대로 나누는 이유는 Kubernetes 클러스터의 역할을 분리해서 이해하기 위해서입니다. `jumpbox`는 관리 작업을 수행하는 작업용 호스트이고, `server`는 Kubernetes control plane 역할을 맡습니다. `node-0`과 `node-1`은 실제 Pod가 실행되는 worker node입니다.

| 이름    | 설명                  | CPU | RAM   | 스토리지 |
| ------- | --------------------- | --- | ----- | -------- |
| jumpbox | 관리용 호스트         | 1   | 512MB | 10GB     |
| server  | Kubernetes 서버       | 1   | 2GB   | 20GB     |
| node-0  | Kubernetes 워커 노드  | 1   | 2GB   | 20GB     |
| node-1  | Kubernetes 워커 노드  | 1   | 2GB   | 20GB     |

머신을 어떤 방식으로 준비할지는 자유입니다. 중요한 것은 각 머신이 위의 사양과 OS 요구사항을 만족해야 한다는 점입니다. 같은 OS와 비슷한 사양으로 맞춰두면 이후 실습에서 패키지 이름, 설정 파일 위치, systemd 동작 차이 때문에 생기는 혼란을 줄일 수 있습니다.

머신 4대를 모두 준비한 뒤 `/etc/os-release` 파일을 확인해서 OS 정보를 검증합니다.

```bash
cat /etc/os-release
```

다음과 비슷한 출력이 보여야 합니다.

```text
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
```

## Multipass로 실습 머신 만들기

현재 Multipass 환경에서는 Debian 12 이미지가 보이지 않았기 때문에 Ubuntu 24.04 LTS 이미지로 실습 머신 4대를 만들었습니다. 원문은 Debian 12 기준이지만, Ubuntu 24.04도 Debian 계열이라 대부분의 실습 흐름을 따라갈 수 있습니다.

Ubuntu 26.04 대신 Ubuntu 24.04를 사용한 이유는 24.04가 현재 더 널리 쓰이는 LTS 버전이고, 실습 중 패키지나 기본 설정 차이로 생길 수 있는 변수를 줄이기 위해서입니다.

먼저 Multipass에서 사용할 수 있는 이미지를 확인했습니다.

```bash
multipass find
```

이미지 목록을 먼저 확인하는 이유는 내 환경에서 실제로 사용할 수 있는 OS 이미지를 기준으로 VM을 만들어야 하기 때문입니다. 문서에 적힌 이미지 이름이 항상 내 Multipass 환경에 그대로 존재하지는 않을 수 있습니다.

Debian 12 이미지가 목록에 없어서 Ubuntu 24.04를 명시해 각 머신을 생성했습니다.

```bash
multipass launch 24.04 --name jumpbox --cpus 1 --memory 512M --disk 10G
multipass launch 24.04 --name server  --cpus 1 --memory 2G   --disk 20G
multipass launch 24.04 --name node-0  --cpus 1 --memory 2G   --disk 20G
multipass launch 24.04 --name node-1  --cpus 1 --memory 2G   --disk 20G
```

`jumpbox`만 메모리를 512MB로 작게 준 이유는 주로 명령 실행, 파일 다운로드, 인증서 생성, 원격 복사 같은 관리 작업만 수행하기 때문입니다. 반면 `server`, `node-0`, `node-1`은 Kubernetes 컴포넌트와 컨테이너를 실행해야 하므로 더 많은 메모리와 디스크를 할당했습니다.

생성이 끝난 뒤 머신 목록을 확인했습니다.

```bash
multipass list
```

목록을 확인하는 이유는 네 머신이 모두 `Running` 상태인지, 그리고 각 머신에 어떤 IP가 할당되었는지 확인하기 위해서입니다. 이후 단계에서는 이 IP를 사용해 SSH 접속, 인증서 생성, Kubernetes 설정 파일 작성, 노드 간 통신을 구성합니다.

현재 실습용 머신은 다음과 같이 구성했습니다.

```text
Name     State    IPv4          Image
jumpbox  Running  192.168.2.10  Ubuntu 24.04 LTS
node-0   Running  192.168.2.8   Ubuntu 24.04 LTS
node-1   Running  192.168.2.9   Ubuntu 24.04 LTS
server   Running  192.168.2.11  Ubuntu 24.04 LTS
```

각 머신의 OS 정보는 다음 명령어로 확인할 수 있습니다.

이 확인은 모든 머신이 같은 계열의 OS로 준비되었는지 검증하는 과정입니다. 한 머신만 다른 버전이거나 예상과 다른 이미지로 만들어진 경우, 이후 실습에서 특정 명령이나 설정이 다르게 동작할 수 있습니다.

```bash
multipass exec jumpbox -- cat /etc/os-release
multipass exec server -- cat /etc/os-release
multipass exec node-0 -- cat /etc/os-release
multipass exec node-1 -- cat /etc/os-release
```

다음 단계: [jumpbox 설정하기](02-jumpbox.md)
