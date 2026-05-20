# n8n Docker CI/CD

GitHub에 push하면 GitHub Actions가 Oracle/Ubuntu 서버에 SSH로 접속해서 n8n Docker Compose를 자동 갱신한다.

## 자동 배포 파일 위치

GitHub Actions 자동 배포 파일은 레포 루트의 아래 경로에 있다.

```text
.github/workflows/deploy.yml
```

이 `n8n-docker` 폴더를 그대로 새 GitHub 리포의 루트로 올리면 위 workflow가 동작한다. `.github/workflows` 폴더를 사용한다.

## 서버 초기 설정

배포 전 Oracle Ubuntu 서버에서 한 번만 실행한다.

### Docker 설치

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

로그아웃 후 다시 SSH 접속해야 `docker` 명령을 sudo 없이 쓸 수 있다.

### swap 2GB

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 로컬 iptables

```bash
for port in 80 443 5678; do
  sudo iptables -A INPUT -p tcp --dport $port -j ACCEPT
done
sudo apt-get install -y iptables-persistent
sudo netfilter-persistent save
```

주의: 위 명령은 Ubuntu 서버 내부의 `iptables` 규칙을 연다. Oracle Cloud 콘솔의 Security List 또는 NSG 인바운드 규칙은 별도로 열어야 한다.

## GitHub Secrets

Repository → Settings → Secrets and variables → Actions → New repository secret에 아래 값을 등록한다.

| Secret | 예시 | 설명 |
|---|---|---|
| `SSH_HOST` | `123.123.123.123` | Oracle 서버 Public IP |
| `SSH_KEY` | `-----BEGIN OPENSSH PRIVATE KEY-----...` | 개인키 전체 내용 |
| `N8N_ENCRYPTION_KEY` | `openssl rand -hex 32` 결과 | n8n credential 암호화 키 |

필수 Secret은 `SSH_HOST`, `SSH_KEY`, `N8N_ENCRYPTION_KEY` 세 개다. SSH 접속은 기본 포트 `22`를 쓰고, 서버 배포 경로는 기본적으로 `~/n8n`을 쓴다. `SSH_USER`는 기본 `ubuntu`다.

workflow는 두 번 검증한다.

1. `.env.example`로 템플릿 문법 검증
2. `.env.example`을 복사한 뒤 `N8N_ENCRYPTION_KEY` Secret 값을 주입해서 실제 배포 환경 검증

서버에 SSH로 들어간 뒤에도 `.env`를 만든 직후 `docker compose --env-file .env config`를 한 번 더 실행한다. `.env.example`은 정상인데 Secret 값이 잘못된 경우도 배포 전에 잡는다.

## N8N_ENCRYPTION_KEY 생성

먼저 암호화 키를 생성한다. 이 값은 n8n credential 암호화에 쓰이므로 **처음 정한 뒤 바꾸지 않는다**.

macOS / Linux / Git Bash:

```bash
openssl rand -hex 32
```

Windows PowerShell (`openssl`가 없는 경우):

```powershell
$bytes = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
-join ($bytes | ForEach-Object { $_.ToString("x2") })
```

생성된 값을 GitHub Secret `N8N_ENCRYPTION_KEY`에 붙여넣는다.

## .env.example 예시: Public IP 실습 모드

`.env.example`에 아래 내용을 넣고 Git에 커밋한다. `N8N_ENCRYPTION_KEY=`는 비워둔다. workflow가 Secret 값을 주입한다.

```env
N8N_HOST=123.123.123.123
N8N_PORT=5678
N8N_PORT_PUBLIC=5678
N8N_BIND_IP=0.0.0.0
N8N_PROTOCOL=http
WEBHOOK_URL=http://123.123.123.123:5678/
GENERIC_TIMEZONE=Asia/Seoul
TZ=Asia/Seoul
N8N_SECURE_COOKIE=false
N8N_ENCRYPTION_KEY=
COMPOSE_PROFILES=
```

## .env.example 예시: 도메인 + HTTPS 모드

도메인 A 레코드가 서버 IP를 가리키고, 서버 방화벽과 OCI Security List에서 80/443이 열려 있어야 한다.

```env
N8N_HOST=n8n.example.com
N8N_PORT=5678
N8N_PORT_PUBLIC=5678
N8N_BIND_IP=127.0.0.1
N8N_PROTOCOL=https
WEBHOOK_URL=https://n8n.example.com/
GENERIC_TIMEZONE=Asia/Seoul
TZ=Asia/Seoul
N8N_SECURE_COOKIE=true
N8N_ENCRYPTION_KEY=
COMPOSE_PROFILES=https
```

## 배포

`main` 브랜치에 push하면 자동 배포된다.

```bash
git add .
git commit -m "add n8n docker cicd"
git push origin main
```

수동 실행도 가능하다.

GitHub → Actions → Deploy n8n to Oracle with Docker → Run workflow

## 서버에서 확인

```bash
cd ~/n8n
docker compose ps
docker compose logs -f n8n
```

## 백업

n8n 데이터는 Docker volume `n8n-docker_n8n_data` 또는 Compose 프로젝트명에 따른 `*_n8n_data`에 저장된다.

```bash
cd ~/n8n
docker compose down
docker run --rm -v n8n-docker_n8n_data:/data -v "$PWD":/backup ubuntu tar czf /backup/n8n_data_backup.tgz /data
docker compose up -d
```
