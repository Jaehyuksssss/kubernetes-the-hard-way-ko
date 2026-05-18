# 컴퓨트 리소스 준비하기

Kubernetes 클러스터는 control plane을 실행할 머신과 실제 컨테이너가 실행될 worker node가 필요합니다. 이번 단계에서는 `jumpbox`에서 `server`, `node-0`, `node-1`을 관리할 수 있도록 머신 정보를 정리하고 SSH 접근, hostname, hosts 설정을 준비합니다.

이 단계를 먼저 하는 이유는 이후 실습 대부분이 `jumpbox`에서 다른 머신으로 명령을 보내거나 파일을 복사하는 방식으로 진행되기 때문입니다. IP와 hostname, SSH 접근이 정리되어 있지 않으면 인증서 생성, 설정 파일 배포, systemd 서비스 설정 단계에서 계속 막히게 됩니다.

## 현재 머신 정보

현재 Multipass VM은 다음 IP로 준비했습니다.

```text
jumpbox  192.168.2.10
server   192.168.2.11
node-0   192.168.2.8
node-1   192.168.2.9
```

`jumpbox`는 관리용 머신이므로 Kubernetes 클러스터 멤버에는 포함하지 않습니다. 실제 클러스터를 구성하는 대상은 `server`, `node-0`, `node-1` 세 대입니다.

## 머신 데이터베이스 만들기

이 튜토리얼에서는 `machines.txt`라는 텍스트 파일을 간단한 머신 데이터베이스처럼 사용합니다. 각 줄은 다음 형식을 따릅니다.

```text
IPV4_ADDRESS FQDN HOSTNAME POD_SUBNET
```

각 필드의 의미는 다음과 같습니다.

- `IPV4_ADDRESS`: 머신의 IPv4 주소
- `FQDN`: fully qualified domain name, 전체 도메인 이름
- `HOSTNAME`: 짧은 호스트 이름
- `POD_SUBNET`: worker node에 할당할 Pod IP 대역

`POD_SUBNET`이 필요한 이유는 Kubernetes가 Pod마다 IP를 하나씩 부여하기 때문입니다. worker node마다 서로 다른 Pod IP 대역을 갖게 해두면, 나중에 Pod 네트워크 라우팅을 구성할 때 어떤 Pod IP가 어느 노드에 있는지 명확해집니다.

`jumpbox` 안에서 저장소 디렉터리로 이동합니다.

```bash
cd /root/kubernetes-the-hard-way-ko
```

현재 환경에 맞춰 `machines.txt`를 생성합니다.

```bash
cat > machines.txt <<EOF
192.168.2.11 server.kubernetes.local server
192.168.2.8 node-0.kubernetes.local node-0 10.200.0.0/24
192.168.2.9 node-1.kubernetes.local node-1 10.200.1.0/24
EOF
```

작성된 내용을 확인합니다.

```bash
cat machines.txt
```

예상 출력은 다음과 같습니다.

```text
192.168.2.11 server.kubernetes.local server
192.168.2.8 node-0.kubernetes.local node-0 10.200.0.0/24
192.168.2.9 node-1.kubernetes.local node-1 10.200.1.0/24
```

`server` 줄에는 `POD_SUBNET`이 없습니다. `server`는 control plane 역할이고, 이 실습에서는 Pod를 실행하는 worker node로 사용하지 않기 때문입니다.

## root SSH 접근 준비

이 튜토리얼은 편의상 `jumpbox`에서 `root@server`, `root@node-0`, `root@node-1`로 접속할 수 있다고 가정합니다. 실제 운영 환경에서는 root SSH 접속을 열어두는 것이 좋지 않지만, 이 실습에서는 명령을 단순하게 유지하고 Kubernetes 구성 자체에 집중하기 위해 사용합니다.

Multipass로 만든 Ubuntu VM은 기본적으로 `root` SSH 접속이 바로 되지 않을 수 있습니다. 그래서 먼저 `jumpbox`의 root SSH 공개키를 만들고, 그 공개키를 세 클러스터 머신의 root 계정에 등록합니다.

### jumpbox에서 SSH 키 생성

아래 명령은 `jumpbox` 안에서 실행합니다.

```bash
ssh-keygen -t ed25519
```

프롬프트가 나오면 기본 경로인 `/root/.ssh/id_ed25519`를 사용하고, passphrase는 비워둡니다.

```text
Enter file in which to save the key (/root/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

passphrase를 비워두는 이유는 이후 스크립트가 여러 머신에 반복적으로 SSH 접속할 때 매번 암호를 입력하지 않기 위해서입니다. 이 역시 실습 편의성을 위한 선택입니다.

생성된 공개키를 확인합니다.

```bash
cat /root/.ssh/id_ed25519.pub
```

예전 문서나 다른 환경에서는 `/root/.ssh/id_rsa.pub`가 나올 수 있습니다. 이 실습에서는 `ed25519` 키를 사용하므로 `/root/.ssh/id_ed25519.pub`를 기준으로 진행합니다. 중요한 것은 파일 이름이 아니라, 공개키 파일의 내용을 각 머신의 `/root/.ssh/authorized_keys`에 등록하는 것입니다.

### Multipass VM에 jumpbox 공개키 등록

이 부분은 `jumpbox` 안이 아니라 로컬 Mac 터미널에서 실행합니다. 이유는 `multipass exec` 명령이 로컬 Mac에서 VM을 직접 제어하는 명령이기 때문입니다.

먼저 `jumpbox`의 root 공개키를 로컬 변수에 담습니다.

```bash
JUMPBOX_ROOT_PUBKEY="$(multipass exec jumpbox -- sudo cat /root/.ssh/id_ed25519.pub)"
```

그다음 `server`, `node-0`, `node-1`의 root 계정에 공개키를 등록합니다.

```bash
for host in server node-0 node-1; do
  multipass exec "$host" -- sudo mkdir -p /root/.ssh
  multipass exec "$host" -- sudo sh -c "echo '$JUMPBOX_ROOT_PUBKEY' >> /root/.ssh/authorized_keys"
  multipass exec "$host" -- sudo chmod 700 /root/.ssh
  multipass exec "$host" -- sudo chmod 600 /root/.ssh/authorized_keys
done
```

공개키를 `authorized_keys`에 넣는 이유는 `jumpbox`의 root private key를 가진 사용자만 각 머신의 root 계정으로 SSH 접속할 수 있게 하기 위해서입니다.

이어서 SSH 서버 설정에서 root 공개키 접속을 허용합니다.

```bash
for host in server node-0 node-1; do
  multipass exec "$host" -- sudo sed -i \
    's/^#*PermitRootLogin.*/PermitRootLogin prohibit-password/' \
    /etc/ssh/sshd_config
  multipass exec "$host" -- sudo systemctl restart ssh
done
```

`PermitRootLogin prohibit-password`는 root 로그인을 허용하되 비밀번호 로그인은 막고, SSH key 로그인만 허용한다는 뜻입니다. 실습 편의성과 최소한의 안전장치 사이의 절충입니다.

## SSH 접속 확인

다시 `jumpbox`로 돌아와서 실행합니다.

```bash
cd /root/kubernetes-the-hard-way-ko
```

IP 주소로 root SSH 접속이 되는지 확인합니다.

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n -o StrictHostKeyChecking=accept-new root@${IP} hostname
done < machines.txt
```

예상 출력은 아직 hostname 설정 전이라 Multipass가 만든 기본 이름이 나올 수 있습니다. 접속 자체가 성공하면 다음 단계로 진행할 수 있습니다.

`-n`을 넣는 이유는 `ssh`가 `machines.txt`의 남은 줄을 표준입력으로 읽어버리는 것을 막기 위해서입니다. 이 옵션이 없으면 첫 번째 머신만 처리하고 반복문이 끝날 수 있습니다.

`StrictHostKeyChecking=accept-new`를 넣은 이유는 처음 접속하는 서버의 host key를 자동으로 `known_hosts`에 등록하기 위해서입니다. 이후 SSH 명령에서는 같은 서버인지 확인하는 데 이 정보가 사용됩니다.

이 옵션을 쓰지 않으면 처음 접속할 때 다음과 같은 host key 확인 질문이 나올 수 있습니다.

```text
Are you sure you want to continue connecting (yes/
no/[fingerprint])?
```

이 경우 `yes`를 입력합니다. 이 과정은 SSH가 처음 보는 서버의 host key를 `known_hosts`에 저장하는 단계입니다.

## Hostname 설정

이제 각 머신의 hostname을 `server`, `node-0`, `node-1`로 설정합니다.

hostname을 맞추는 이유는 Kubernetes 컴포넌트들이 노드를 식별할 때 hostname을 사용하기 때문입니다. 예를 들어 worker node의 `kubelet`은 클러스터에 등록될 때 자신의 이름으로 hostname을 사용합니다. 또한 이후 설정 파일과 인증서에서도 `server`, `node-0`, `node-1` 같은 이름을 계속 사용합니다.

`jumpbox`에서 다음 명령을 실행합니다.

```bash
while read IP FQDN HOST SUBNET; do
    CMD="sed -i 's/^127.0.1.1.*/127.0.1.1\t${FQDN} ${HOST}/' /etc/hosts"
    ssh -n root@${IP} "$CMD"
    ssh -n root@${IP} hostnamectl set-hostname ${HOST}
    ssh -n root@${IP} systemctl restart systemd-hostnamed
done < machines.txt
```

설정 결과를 확인합니다.

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname --fqdn
done < machines.txt
```

예상 출력은 다음과 같습니다.

```text
server.kubernetes.local
node-0.kubernetes.local
node-1.kubernetes.local
```

## hosts 파일 만들기

이제 각 머신 이름을 IP로 해석할 수 있도록 `hosts` 파일을 만듭니다.

DNS 서버를 따로 구성하지 않고 `/etc/hosts`를 사용하는 이유는 실습을 단순하게 유지하기 위해서입니다. 작은 실습 클러스터에서는 각 머신의 이름과 IP를 `/etc/hosts`에 직접 넣어도 충분합니다.

`jumpbox`에서 `hosts` 파일을 생성합니다.

```bash
echo "" > hosts
echo "# Kubernetes The Hard Way" >> hosts
```

`machines.txt`를 읽어서 host entry를 추가합니다.

```bash
while read IP FQDN HOST SUBNET; do
    ENTRY="${IP} ${FQDN} ${HOST}"
    echo $ENTRY >> hosts
done < machines.txt
```

내용을 확인합니다.

```bash
cat hosts
```

예상 출력은 다음과 같습니다.

```text

# Kubernetes The Hard Way
192.168.2.11 server.kubernetes.local server
192.168.2.8 node-0.kubernetes.local node-0
192.168.2.9 node-1.kubernetes.local node-1
```

## jumpbox의 `/etc/hosts`에 추가하기

`jumpbox`에서 `server`, `node-0`, `node-1` 같은 이름으로 접속할 수 있도록 `hosts` 내용을 `/etc/hosts`에 추가합니다.

```bash
cat hosts >> /etc/hosts
```

확인합니다.

```bash
cat /etc/hosts
```

이제 `jumpbox`에서 hostname만으로 SSH 접속이 되는지 확인합니다.

```bash
for host in server node-0 node-1; do
  ssh root@${host} hostname
done
```

예상 출력은 다음과 같습니다.

```text
server
node-0
node-1
```

## 원격 머신의 `/etc/hosts`에도 추가하기

마지막으로 `server`, `node-0`, `node-1` 서로도 이름으로 통신할 수 있도록 각 머신의 `/etc/hosts`에 같은 내용을 추가합니다.

이 작업이 필요한 이유는 Kubernetes 컴포넌트들이 서로 IP만으로 통신하지 않고 hostname 또는 인증서에 들어간 이름을 기준으로 접근하는 경우가 있기 때문입니다. 모든 머신이 같은 이름 해석 정보를 갖고 있으면 이후 인증서와 설정을 다룰 때 혼란이 줄어듭니다.

`jumpbox`에서 다음 명령을 실행합니다.

```bash
while read IP FQDN HOST SUBNET; do
  scp hosts root@${HOST}:~/
  ssh -n root@${HOST} "cat hosts >> /etc/hosts"
done < machines.txt
```

각 머신에서 hostname으로 다른 머신을 찾을 수 있는지 확인합니다.

```bash
for host in server node-0 node-1; do
  ssh root@${host} "getent hosts server node-0 node-1"
done
```

여기까지 완료하면 `jumpbox`와 Kubernetes 클러스터에 들어갈 세 머신 모두에서 `server`, `node-0`, `node-1`이라는 이름을 사용할 수 있습니다.

# 정리

이번 단계에서는 `jumpbox`에서 `server`, `node-0`, `node-1`을 관리할 수 있도록 머신 정보, SSH 접근, hostname, `/etc/hosts` 설정을 완료했습니다.

현재 Multipass VM의 IP는 다음과 같았습니다.

```text
jumpbox  192.168.2.10
server   192.168.2.11
node-0   192.168.2.8
node-1   192.168.2.9
```

`jumpbox` 안에서 `machines.txt`를 다음 내용으로 만들었습니다.

```text
192.168.2.11 server.kubernetes.local server
192.168.2.8 node-0.kubernetes.local node-0 10.200.0.0/24
192.168.2.9 node-1.kubernetes.local node-1 10.200.1.0/24
```

`server`에는 `POD_SUBNET`을 넣지 않았습니다. 이 실습에서 `server`는 control plane 역할만 하고, 실제 Pod는 `node-0`, `node-1`에서 실행하기 때문입니다.

`jumpbox`에서는 `ssh-keygen -t ed25519`로 root SSH 키를 만들었습니다. 실제 생성된 공개키 파일은 다음 경로였습니다.

```text
/root/.ssh/id_ed25519.pub
```

처음에는 `/root/.ssh/id_rsa.pub`를 확인했지만, 현재 환경에서는 RSA 키가 아니라 ed25519 키가 생성되어 있었기 때문에 `/root/.ssh/id_ed25519.pub`를 사용했습니다.

`multipass` 명령은 `jumpbox` 안이 아니라 로컬 Mac 터미널에서 실행해야 합니다. 그래서 로컬 Mac에서 `jumpbox`의 root 공개키를 읽고, `server`, `node-0`, `node-1`의 root 계정에 등록했습니다.

```bash
JUMPBOX_ROOT_PUBKEY="$(multipass exec jumpbox -- sudo cat /root/.ssh/id_ed25519.pub)"

for host in server node-0 node-1; do
  multipass exec "$host" -- sudo mkdir -p /root/.ssh
  multipass exec "$host" -- sudo sh -c "echo '$JUMPBOX_ROOT_PUBKEY' >> /root/.ssh/authorized_keys"
  multipass exec "$host" -- sudo chmod 700 /root/.ssh
  multipass exec "$host" -- sudo chmod 600 /root/.ssh/authorized_keys
done
```

root SSH 접속을 위해 각 머신의 SSH 설정도 로컬 Mac에서 변경했습니다.

```bash
for host in server node-0 node-1; do
  multipass exec "$host" -- sudo sed -i \
    's/^#*PermitRootLogin.*/PermitRootLogin prohibit-password/' \
    /etc/ssh/sshd_config
  multipass exec "$host" -- sudo systemctl restart ssh
done
```

`jumpbox`에서 IP 기반 root SSH 접속을 확인했습니다.

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n -o StrictHostKeyChecking=accept-new root@${IP} hostname
done < machines.txt
```

확인 결과는 다음과 같았습니다.

```text
server
node-0
node-1
```

여기서 `-n` 옵션이 중요했습니다. `ssh`가 `machines.txt`의 남은 줄을 표준입력으로 읽어버리면 첫 번째 머신만 처리하고 반복문이 끝날 수 있기 때문입니다.

각 머신의 hostname을 설정했고, 확인 결과는 다음과 같았습니다.

```text
server.kubernetes.local
node-0.kubernetes.local
node-1.kubernetes.local
```

`hosts` 파일은 다음 내용으로 생성했습니다.

```text

# Kubernetes The Hard Way
192.168.2.11 server.kubernetes.local server
192.168.2.8 node-0.kubernetes.local node-0
192.168.2.9 node-1.kubernetes.local node-1
```

이 내용을 먼저 `jumpbox`의 `/etc/hosts`에 추가한 뒤, hostname 기반 SSH 접속을 확인했습니다.

```bash
for host in server node-0 node-1; do
  ssh -o StrictHostKeyChecking=accept-new root@${host} hostname
done
```

확인 결과는 다음과 같았습니다.

```text
server
node-0
node-1
```

마지막으로 같은 `hosts` 파일을 `server`, `node-0`, `node-1`에도 복사하고 각 머신의 `/etc/hosts`에 추가했습니다.

```bash
while read IP FQDN HOST SUBNET; do
  scp hosts root@${HOST}:~/
  ssh -n root@${HOST} "cat hosts >> /etc/hosts"
done < machines.txt
```

각 머신에서 이름 해석을 확인한 결과, 세 머신 모두 `server`, `node-0`, `node-1`을 IP로 해석할 수 있었습니다. 출력에 `127.0.1.1` 항목이 함께 보였지만, 각 머신 자신의 hostname이 로컬 주소에도 등록되어 있기 때문에 정상입니다.

따라서 `03-compute-resources.md` 단계는 완료되었습니다.

다음 단계: [CA 준비와 TLS 인증서 생성하기](04-certificate-authority.md)
