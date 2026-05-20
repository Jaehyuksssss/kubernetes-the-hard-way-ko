# etcd 클러스터 부트스트랩하기

Kubernetes 컴포넌트들은 대부분 상태를 직접 저장하지 않습니다. 클러스터 상태는 `etcd`에 저장됩니다. 이번 단계에서는 `server` 머신에 단일 노드 etcd 클러스터를 구성합니다.

etcd는 Kubernetes의 핵심 저장소입니다. 예를 들어 다음과 같은 정보들이 etcd에 저장됩니다.

- Node 정보
- Pod 상태
- Service 정보
- ConfigMap
- Secret
- 클러스터 전체 설정과 상태

즉, Kubernetes API 서버는 클러스터 상태를 읽고 쓸 때 etcd를 사용합니다. 그래서 control plane을 띄우기 전에 etcd가 먼저 실행되어 있어야 합니다.

## 사전 준비

먼저 `jumpbox`에서 `server`로 etcd 바이너리와 systemd unit 파일을 복사합니다.

이 명령은 `jumpbox`에서 실행합니다.

```bash
scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```

각 파일의 의미는 다음과 같습니다.

- `etcd`: etcd 서버 실행 파일
- `etcdctl`: etcd를 확인하고 조작하는 CLI 도구
- `etcd.service`: etcd를 systemd 서비스로 실행하기 위한 unit 파일

복사가 끝나면 `server`에 접속합니다.

```bash
ssh root@server
```

이후 명령은 반드시 `server` 머신에서 실행합니다. 프롬프트가 다음과 같은지 확인합니다.

```text
root@server:~#
```

`root@jumpbox` 상태에서 server용 명령을 실행하면 파일을 찾지 못하거나, jumpbox에 잘못된 설정 디렉터리를 만들 수 있습니다.

## etcd 바이너리 설치

`server`에서 `etcd`와 `etcdctl`을 `/usr/local/bin`으로 옮깁니다.

```bash
{
  mv etcd etcdctl /usr/local/bin/
}
```

`/usr/local/bin`으로 옮기는 이유는 어느 디렉터리에서든 `etcd`, `etcdctl` 명령을 실행할 수 있게 하기 위해서입니다.

설치 확인:

```bash
which etcd
which etcdctl
etcd --version
etcdctl version
```

## etcd 서버 설정

etcd 설정 파일과 데이터 디렉터리를 준비합니다.

```bash
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-api-server.key kube-api-server.crt \
    /etc/etcd/
}
```

각 디렉터리의 의미는 다음과 같습니다.

- `/etc/etcd`: etcd가 사용할 인증서와 설정 파일을 두는 위치
- `/var/lib/etcd`: etcd 데이터가 저장되는 위치

`chmod 700 /var/lib/etcd`를 하는 이유는 etcd 데이터 디렉터리를 root만 접근할 수 있게 제한하기 위해서입니다. etcd에는 Kubernetes 클러스터 상태와 Secret 관련 데이터가 저장될 수 있으므로 권한을 좁게 두는 것이 좋습니다.

`ca.crt`, `kube-api-server.key`, `kube-api-server.crt`를 `/etc/etcd`로 복사하는 이유는 API 서버와 etcd가 통신할 때 필요한 TLS 재료를 etcd 쪽에서도 사용할 수 있게 하기 위해서입니다.

## systemd unit 설치

복사해둔 `etcd.service` 파일을 systemd가 읽는 위치로 옮깁니다.

```bash
mv etcd.service /etc/systemd/system/
```

이 파일은 etcd를 어떤 명령과 옵션으로 실행할지 정의합니다. systemd unit으로 등록해두면 etcd를 직접 명령어로 실행하지 않고 서비스처럼 관리할 수 있습니다.

## etcd 서버 시작

systemd 설정을 다시 읽고, etcd 서비스를 활성화한 뒤 시작합니다.

```bash
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```

각 명령의 의미는 다음과 같습니다.

- `systemctl daemon-reload`: 새로 추가한 unit 파일을 systemd가 다시 읽게 합니다.
- `systemctl enable etcd`: 부팅 시 etcd가 자동 시작되도록 등록합니다.
- `systemctl start etcd`: 지금 etcd 서비스를 시작합니다.

서비스 상태를 확인합니다.

```bash
systemctl status etcd --no-pager
```

정상이라면 다음과 비슷하게 보입니다.

```text
Active: active (running)
```

## 검증

etcd 클러스터 멤버를 확인합니다.

```bash
etcdctl member list
```

예상 출력은 다음과 비슷합니다.

```text
6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false
```

이 출력은 단일 노드 etcd 클러스터가 실행 중이라는 뜻입니다.

- `started`: etcd 멤버가 정상 시작됨
- `controller`: etcd 멤버 이름
- `127.0.0.1:2380`: etcd peer 통신 주소
- `127.0.0.1:2379`: etcd client 요청 주소

## 내 실습 기록

이번 단계에서는 `jumpbox`에서 `server`로 필요한 파일을 복사하고, `server`에서 etcd를 systemd 서비스로 실행했습니다.

먼저 `jumpbox`에서 다음 파일을 `server`로 복사했습니다.

```bash
scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```

그다음 `server`에 접속했습니다.

```bash
ssh root@server
```

처음에 `jumpbox`에서 `mv etcd etcdctl /usr/local/bin/`을 실행해서 다음 에러가 발생했습니다.

```text
mv: cannot stat 'etcd': No such file or directory
mv: cannot stat 'etcdctl': No such file or directory
```

이 에러의 이유는 `etcd`, `etcdctl` 파일이 `server`의 `/root`에 복사되어 있었는데, 명령을 `jumpbox`에서 실행했기 때문입니다. 이 명령은 `root@server:~#` 상태에서 실행해야 합니다.

`server`에 접속한 뒤 etcd 바이너리를 설치했습니다.

```bash
mv etcd etcdctl /usr/local/bin/
```

설치 확인 결과는 다음과 같았습니다.

```text
/usr/local/bin/etcd
/usr/local/bin/etcdctl
etcd Version: 3.6.0-rc.3
etcdctl version: 3.6.0-rc.3
API version: 3.6
```

중간에 `server` 안에서 다시 `scp downloads/controller/etcd ...`를 실행했지만, 이 명령은 `jumpbox`에서 실행해야 하는 복사 명령이므로 실패했습니다.

```text
scp: stat local "downloads/controller/etcd": No such file or directory
```

이미 etcd 바이너리 설치는 완료된 상태였기 때문에 이 에러는 무시하고 진행했습니다.

또 `server` 안에서 `ssh root@server`를 실행했을 때 다음 에러가 있었습니다.

```text
root@server: Permission denied (publickey).
```

이미 `server` 안에 있는 상태에서 자기 자신에게 SSH 접속을 시도한 것이므로 이 단계에는 필요 없는 명령이었습니다.

그다음 `server`에서 etcd 설정 디렉터리를 만들고 인증서를 복사했습니다.

```bash
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-api-server.key kube-api-server.crt \
    /etc/etcd/
}
```

`etcd.service`를 systemd unit 위치로 옮겼습니다.

```bash
mv etcd.service /etc/systemd/system/
```

etcd 서비스를 시작했습니다.

```bash
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```

서비스 상태 확인 결과, etcd가 정상 실행 중이었습니다.

```text
Active: active (running)
```

마지막으로 etcd 멤버 목록을 확인했습니다.

```bash
etcdctl member list
```

결과는 다음과 같았습니다.

```text
6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false
```

따라서 `07-bootstrapping-etcd.md` 단계는 완료되었습니다.

다음 단계: [Kubernetes Control Plane 부트스트랩하기](08-bootstrapping-kubernetes-controllers.md)
