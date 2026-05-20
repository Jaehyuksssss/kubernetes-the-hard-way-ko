# 데이터 암호화 설정과 키 생성하기

Kubernetes는 클러스터 상태, 애플리케이션 설정, Secret 같은 데이터를 저장합니다. 이 데이터들은 최종적으로 `etcd`에 저장됩니다.

이번 단계에서는 Kubernetes Secret 데이터를 etcd에 저장할 때 암호화하기 위한 encryption key와 encryption config 파일을 생성합니다. 명령은 모두 `jumpbox`의 저장소 디렉터리에서 실행합니다.

```bash
cd /root/kubernetes-the-hard-way-ko
```

## 왜 암호화 설정이 필요한가

Kubernetes의 Secret은 이름 그대로 민감한 값을 저장할 때 사용합니다. 예를 들어 다음과 같은 데이터가 Secret에 들어갈 수 있습니다.

- 데이터베이스 비밀번호
- API token
- TLS private key
- Docker registry 인증 정보

기본적으로 Kubernetes 리소스는 API 서버를 통해 생성되고, 그 상태는 etcd에 저장됩니다. Secret도 예외가 아닙니다. 그래서 etcd 디스크에 접근할 수 있는 사람이 Secret 내용을 그대로 읽을 수 있다면 위험합니다.

encryption config는 API 서버에게 다음과 같은 규칙을 알려줍니다.

```text
Secret 리소스를 etcd에 저장할 때는 이 key로 암호화해서 저장하라.
Secret 리소스를 읽을 때는 이 key로 복호화해서 API 응답에 사용하라.
```

즉, 이 단계는 Kubernetes 자체 컴포넌트를 실행하기 전 Secret 저장 방식을 미리 정하는 작업입니다.

## 암호화 키 생성

먼저 32바이트 랜덤 값을 만들고 base64로 인코딩해서 `ENCRYPTION_KEY` 환경 변수에 저장합니다.

```bash
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

명령의 의미는 다음과 같습니다.

- `head -c 32 /dev/urandom`: 운영체제의 랜덤 소스에서 32바이트를 읽습니다.
- `base64`: 바이너리 값을 YAML에 넣기 좋은 문자열 형태로 바꿉니다.
- `export ENCRYPTION_KEY=...`: 이후 `envsubst`가 사용할 환경 변수로 등록합니다.

생성된 값을 확인하고 싶다면 다음 명령을 사용할 수 있습니다.

```bash
echo ${ENCRYPTION_KEY}
```

이 값은 Secret 데이터를 암호화하는 데 쓰이므로 민감한 값입니다. 실습에서는 터미널에서 확인해도 괜찮지만, 실제 운영 환경에서는 노출되지 않도록 관리해야 합니다.

## encryption config 파일 생성

저장소에는 템플릿 파일이 이미 있습니다.

```bash
cat configs/encryption-config.yaml
```

이 템플릿 안에는 `${ENCRYPTION_KEY}` 같은 환경 변수 자리가 들어 있습니다. `envsubst`는 이 값을 현재 셸의 `ENCRYPTION_KEY` 값으로 치환해서 실제 설정 파일을 만듭니다.

```bash
envsubst < configs/encryption-config.yaml \
  > encryption-config.yaml
```

생성된 파일을 확인합니다.

```bash
cat encryption-config.yaml
```

이 파일은 Kubernetes API 서버가 시작될 때 `--encryption-provider-config` 옵션으로 읽게 됩니다. 아직 API 서버를 실행하지는 않았지만, 뒤에서 control plane을 구성할 때 이 파일이 필요합니다.

## server로 encryption config 복사

Kubernetes API 서버는 `server` 머신에서 실행될 예정입니다. 따라서 생성한 `encryption-config.yaml` 파일을 `server`로 복사합니다.

```bash
scp encryption-config.yaml root@server:~/
```

복사되었는지 확인합니다.

```bash
ssh root@server "ls -l ~/encryption-config.yaml"
```

여기까지 완료하면 API 서버가 Secret 데이터를 etcd에 암호화해서 저장할 준비가 된 것입니다.

## 내 실습 기록

이 섹션은 실습을 완료한 뒤 실제 결과를 정리하는 곳입니다.

다음 단계: [etcd 클러스터 부트스트랩하기](07-bootstrapping-etcd.md)
