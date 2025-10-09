# Vagrantfile (VirtualBox, Ubuntu 22.04)
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.box_version = "20241002.0.0"
  config.vm.hostname = "lab-CI-CD"
  config.vm.network "private_network", ip: "192.168.56.102"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "lab-CI-CD"
    vb.memory = 2048
    vb.cpus   = 1
  end

  config.vm.provision "shell",
    privileged: true,
    env: {"HOST_IP" => "192.168.56.102"},
    inline: <<'SHELL'
set -euxo pipefail
export DEBIAN_FRONTEND=noninteractive

# 1) 기본 패키지 + Docker 설치 (공식 리포지토리)
apt-get update
apt-get install -y ca-certificates curl gnupg lsb-release net-tools git
apt-get remove -y docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc || true
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release; echo $UBUNTU_CODENAME) stable" \
  > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin
usermod -aG docker vagrant

# 2) 공용 네트워크/볼륨
docker network create corpnet || true
mkdir -p /srv/{db,gitea,jenkins}/data
chown -R vagrant:vagrant /srv

# 3) MariaDB (문서의 환경변수/이미지 그대로)
export MYSQL_ROOT_PASSWORD='rootpw'
export MYSQL_DB='test'
export MYSQL_USER='testuser'
export MYSQL_PASS='testpw'
docker rm -f db || true
docker run -d --name db --network corpnet \
  -e MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD \
  -e MYSQL_DATABASE=$MYSQL_DB \
  -e MYSQL_USER=$MYSQL_USER \
  -e MYSQL_PASSWORD=$MYSQL_PASS \
  -v /srv/db/data:/var/lib/mysql \
  mariadb:11

# 4) Gitea (문서의 포트/루트URL/웹훅 허용 목록 적용)
docker rm -f gitea || true
docker run -d --name gitea --network corpnet \
  -p 13000:3000 -p 1222:22 \
  -v /srv/gitea/data:/data \
  -e GITEA__server__DOMAIN="$HOST_IP" \
  -e GITEA__server__ROOT_URL="http://$HOST_IP:13000/" \
  -e GITEA__server__PROTOCOL="http" \
  -e GITEA__server__HTTP_PORT=3000 \
  -e GITEA__session__COOKIE_SECURE="false" \
  -e GITEA__webhook__ALLOWED_HOST_LIST="jenkins,jenkins:8080,$HOST_IP,$HOST_IP:18081" \
  gitea/gitea:latest

# 5) Jenkins (도커 빌드/배포 권한을 위해 docker.sock + 바이너리 마운트)
docker rm -f jenkins || true
docker run -d --name jenkins --network corpnet \
  -p 18081:8080 -p 50000:50000 \
  -v /srv/jenkins:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/bin/docker:/usr/bin/docker \
  -u root \
  jenkins/jenkins:lts

# 초기 관리자 비밀번호 추출
# 최대 180초(60회 × 3초) 대기: 파일이 '존재 && 비어있지 않음(-s)'일 때만 저장
PASS_OUT="/vagrant/jenkins_initial_admin_password.txt"
rm -f "$PASS_OUT"

for i in $(seq 1 60); do
  if docker exec jenkins sh -lc 'test -s /var/jenkins_home/secrets/initialAdminPassword'; then
    docker exec jenkins sh -lc 'cat /var/jenkins_home/secrets/initialAdminPassword' > "$PASS_OUT"
    break
  fi
  sleep 3
done


# 6) 샘플 애플리케이션 코드 & Jenkinsfile
su - vagrant -c 'mkdir -p ~/cicd/web ~/cicd/was'
cat <<'DOCK' >/home/vagrant/cicd/web/Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
COPY nginx.conf /etc/nginx/nginx.conf
DOCK

cat <<'CONFN' >/home/vagrant/cicd/web/nginx.conf
worker_processes auto;
events { worker_connections 1024; }
http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;
  sendfile on;
  upstream php_fpm { server was:9000; }
  server {
    listen 80; server_name _;
    root /usr/share/nginx/html; index index.html;
    location ~ \.php$ {
      include fastcgi_params;
      fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
      fastcgi_pass php_fpm;
    }
  }
}
CONFN

cat <<'INDEX' >/home/vagrant/cicd/web/index.html
<!doctype html><html lang="ko"><head><meta charset="utf-8"><title>회사 포털</title></head>
<body><h1>우리 회사 포털</h1><p>게시판으로 이동:</p><p><a href="/index.php">게시판 바로가기</a></p></body></html>
INDEX

cat <<'DOCK2' >/home/vagrant/cicd/was/Dockerfile
FROM php:8.2-fpm-alpine
RUN docker-php-ext-install mysqli pdo pdo_mysql
WORKDIR /var/www/html
COPY index.php /var/www/html/index.php
DOCK2

cat <<'PHP' >/home/vagrant/cicd/was/index.php
<?php
$host = getenv('DB_HOST') ?: 'db';
$db   = getenv('DB_NAME') ?: 'test';
$user = getenv('DB_USER') ?: 'testuser';
$pass = getenv('DB_PASS') ?: 'testpw';
$mysqli = new mysqli($host, $user, $pass, $db);
if ($mysqli->connect_error) { die('DB 연결 실패: ' . $mysqli->connect_error); }
$mysqli->query("CREATE TABLE IF NOT EXISTS posts (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(200) NOT NULL,
  body TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
if (isset($_GET['health'])) {
  mysqli_report(MYSQLI_REPORT_OFF);
  $m = @new mysqli($host, $user, $pass, $db);
  if ($m && !$m->connect_errno) { header('Content-Type: text/plain'); echo 'OK'; exit; }
  http_response_code(500); header('Content-Type: text/plain'); echo 'FAIL'; exit;
}
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
  $title = $_POST['title'] ?? '';
  $body  = $_POST['body'] ?? '';
  if ($title && $body) {
    $stmt = $mysqli->prepare('INSERT INTO posts(title, body) VALUES (?, ?)');
    $stmt->bind_param('ss', $title, $body);
    $stmt->execute();
    header('Location: /index.php'); exit;
  }
}
$res = $mysqli->query('SELECT id, title, body, created_at FROM posts ORDER BY id DESC LIMIT 50');
?><!doctype html><html lang="ko"><head><meta charset="utf-8"><title>게시판 (WAS)</title></head>
<body><h1>간단 게시판</h1>
<form method="POST"><p>제목: <input name="title" required></p>
<p>내용: <textarea name="body" required></textarea></p><button type="submit">등록</button></form>
<hr><h2>목록</h2><ul>
<?php while($row = $res->fetch_assoc()): ?>
<li><strong><?=htmlspecialchars($row['title'])?></strong> (<?=$row['created_at']?>)<br>
<?=nl2br(htmlspecialchars($row['body']))?></li><br>
<?php endwhile; ?></ul></body></html>
PHP

cat <<'JENK' >/home/vagrant/cicd/Jenkinsfile
pipeline {
  agent any
  options { disableConcurrentBuilds(); timestamps() }

  environment {
    NETWORK         = 'corpnet'
    WEB_IMAGE_NAME  = 'test-nginx'
    WAS_IMAGE_NAME  = 'test-php'
    DB_HOST         = 'db'
    DB_NAME         = 'test'
    DB_USER         = 'testuser'
    DB_PASS         = 'testpw'
    APP_PORT        = '18080'
  }

  stages {
    stage('Detect Changes') {
      steps {
        script {
          // 현재/직전 커밋 태그 계산 (최초 실행 대비 기본값 처리)
          env.TAG      = (env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : "dev-${env.BUILD_NUMBER}")
          env.PREV_TAG = sh(script: 'git rev-parse --short=7 HEAD~1 2>/dev/null || true', returnStdout: true).trim()

          // 변경된 최상위 디렉터리 집합 (없으면 전체 배포)
          def changed = sh(script: 'git diff --name-only HEAD~1 HEAD 2>/dev/null | cut -d/ -f1 | sort -u', returnStdout: true).trim()
          def set = (changed ? changed.split("\\s+") as Set : ['web','was'] as Set)

          env.DO_BUILD_WEB = set.contains('web') ? '1' : '0'
          env.DO_BUILD_WAS = set.contains('was') ? '1' : '0'

          echo "TAG=${env.TAG}, PREV_TAG=${env.PREV_TAG}, CHANGED=${set}"
        }
      }
    }

    stage('Build Web Image') {
      when { environment name: 'DO_BUILD_WEB', value: '1' }
      steps {
        sh """
          docker build -t ${WEB_IMAGE_NAME}:${TAG} ./web
          docker tag ${WEB_IMAGE_NAME}:${TAG} ${WEB_IMAGE_NAME}:latest
        """
      }
    }

    stage('Build WAS Image') {
      when { environment name: 'DO_BUILD_WAS', value: '1' }
      steps {
        sh """
          docker build -t ${WAS_IMAGE_NAME}:${TAG} ./was
          docker tag ${WAS_IMAGE_NAME}:${TAG} ${WAS_IMAGE_NAME}:latest
        """
      }
    }

    stage('Deploy') {
      steps {
        sh """
          docker network inspect ${NETWORK} >/dev/null 2>&1 || docker network create ${NETWORK}

          if [ "${DO_BUILD_WAS}" = "1" ]; then
            docker rm -f was >/dev/null 2>&1 || true
            docker run -d --name was --restart unless-stopped --network ${NETWORK} \\
              -e DB_HOST=${DB_HOST} -e DB_NAME=${DB_NAME} -e DB_USER=${DB_USER} -e DB_PASS=${DB_PASS} \\
              ${WAS_IMAGE_NAME}:${TAG}
          fi

          if [ "${DO_BUILD_WEB}" = "1" ]; then
            docker rm -f web >/dev/null 2>&1 || true
            docker run -d --name web --restart unless-stopped --network ${NETWORK} -p ${APP_PORT}:80 \\
              ${WEB_IMAGE_NAME}:${TAG}
          fi
        """
      }
    }
  }

  post {
    failure {
      script {
        echo "Deploy failed. Try rollback to PREV_TAG=${env.PREV_TAG}"
        if (env.PREV_TAG) {
          sh """
            if [ "${DO_BUILD_WEB}" = "1" ] && docker image inspect ${WEB_IMAGE_NAME}:${PREV_TAG} >/dev/null 2>&1; then
              docker rm -f web >/dev/null 2>&1 || true
              docker run -d --name web --restart unless-stopped --network ${NETWORK} -p ${APP_PORT}:80 \\
                ${WEB_IMAGE_NAME}:${PREV_TAG}
            fi
            if [ "${DO_BUILD_WAS}" = "1" ] && docker image inspect ${WAS_IMAGE_NAME}:${PREV_TAG} >/dev/null 2>&1; then
              docker rm -f was >/dev/null 2>&1 || true
              docker run -d --name was --restart unless-stopped --network ${NETWORK} \\
                -e DB_HOST=${DB_HOST} -e DB_NAME=${DB_NAME} -e DB_USER=${DB_USER} -e DB_PASS=${DB_PASS} \\
                ${WAS_IMAGE_NAME}:${PREV_TAG}
            fi
          """
        } else {
          echo "No PREV_TAG available for rollback."
        }
      }
    }
  }
}
JENK

chown -R vagrant:vagrant /home/vagrant/cicd
SHELL
end
