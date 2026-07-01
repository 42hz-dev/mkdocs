# Docker 개발환경 구성

> PHP 7.2 + Nginx + MariaDB 개발환경을 Docker Compose로 구성하는 방법입니다.

---

# 시스템 구성

```text
docker/

├── docker-compose.yml
├── php/
│   └── Dockerfile
├── nginx/
│   └── default.conf
├── db/
│   └── conf/
│       └── my.cnf
├── www/
│   └── (PHP Source)
└── scripts/
    └── import_db.sh
```

---

# Dockerfile

PHP 7.2 FPM 이미지를 기반으로 필요한 PHP Extension을 설치합니다.

```dockerfile
FROM php:7.2-fpm

# Debian Buster Repository 변경 (404 해결)
RUN sed -i 's|deb.debian.org|archive.debian.org|g' /etc/apt/sources.list \
    && sed -i 's|security.debian.org|archive.debian.org|g' /etc/apt/sources.list \
    && sed -i '/buster-updates/d' /etc/apt/sources.list

RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libzip-dev \
    libxml2-dev \
    libonig-dev \
    libcurl4-openssl-dev \
    libssl-dev \
    libldap2-dev \
    libpq-dev \
    libsqlite3-dev \
    libsodium-dev \
    libxslt1-dev \
    libtidy-dev \
    libpspell-dev \
    libgmp-dev \
    libbz2-dev \
    libreadline-dev \
    libkrb5-dev \
    unzip \
    git \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include --with-jpeg-dir=/usr/include \
    && docker-php-ext-install -j$(nproc) \
        bcmath \
        calendar \
        exif \
        ftp \
        gd \
        mysqli \
        pdo_mysql \
        sockets \
        soap \
        pcntl \
        shmop \
        sysvmsg \
        sysvsem \
        sysvshm \
        zip \
        opcache \
        intl \
        xsl \
        tidy \
        gettext \
        gmp \
        bz2

WORKDIR /var/www/html
```

---

# Docker Compose

Docker Compose를 이용하여 Nginx, PHP, MariaDB를 동시에 실행합니다.

```yaml
services:
  nginx:
    image: nginx:1.25.4
    container_name: your-nginx
    ports:
      - "80:80"
    volumes:
      - ./www:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
    networks:
      - your_network

  php:
    build: ./php
    container_name: your-php
    volumes:
      - ./www:/var/www/html
    networks:
      - your_network

  db:
    image: mariadb:10.6
    container_name: your-db
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: YOUR_DATABASE
      MYSQL_USER: YOUR_USER
      MYSQL_PASSWORD: YOUR_PASSWORD
      TZ: Asia/Seoul
    ports:
      - "3306:3306"
    volumes:
      - ./db/conf/my.cnf:/etc/mysql/conf.d/my.cnf
    networks:
      - your_network

volumes:
  web_db:

networks:
  your_network:
    external: true
```

---

# Docker Network 생성

외부 네트워크를 사용하는 경우 미리 생성합니다.

```bash
docker network create your_network
```

---

# 개발환경 실행

컨테이너 생성 및 실행

```bash
docker compose up -d
```

실행 중인 컨테이너 확인

```bash
docker ps
```

종료

```bash
docker compose down
```

---

# DB Import Script

운영 DB Dump를 개발환경으로 Import하기 위한 자동화 스크립트입니다.

```bash
#!/bin/bash

DB_USER="user"
DB_PASS="pass"
DB_NAME="database_name"
CONTAINER="container_name"

BASE_DIR="$HOME/Downloads"
TODAY=$(date +%Y%m%d)
FILENAME="YourFileName"
GUNZIP_FILENAME="YourGunZipFileName.gz"
#TODAY=20260608

SQL_FILE=$(ls ${BASE_DIR}/${TODAY}${FILENAME} 2>/dev/null | head -n 1)
GZ_FILE=$(ls ${BASE_DIR}/${TODAY}${GUNZIP_FILENAME} 2>/dev/null | head -n 1)

if [ -z "$SQL_FILE" ] && [ -z "$GZ_FILE" ]; then
    echo "❌ 오늘 날짜 dump 파일 없음"
    exit 1
fi

# gz만 있으면 압축 해제
if [ -z "$SQL_FILE" ] && [ ! -z "$GZ_FILE" ]; then
    echo "📦 Dump File: $GZ_FILE"
    echo "🗜 압축 해제 중..."
    gunzip -kf "$GZ_FILE"
    SQL_FILE="${GZ_FILE%.gz}"
    echo "✅ 압축 해제 완료"
else
    echo "📄 SQL 파일 사용: $SQL_FILE"
fi

TMP_SQL="/tmp/import.sql"
LOG_FILE="/tmp/import.log"

echo "🧹 SQL 정리 중..."

{
echo "USE \`$DB_NAME\`;"
grep -viE "^(create database|drop database|use )" "$SQL_FILE"
} > "$TMP_SQL"

echo "✅ SQL 정리 완료"

echo ""
echo "🗄 DB 초기화 시작..."

echo -n "기존 DB 삭제... "
docker exec $CONTAINER mysql -u$DB_USER -p$DB_PASS -e "DROP DATABASE IF EXISTS \`$DB_NAME\`;" >/dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✅ 성공"
else
    echo "❌ 실패"
    exit 1
fi

echo -n "DB 생성... "
docker exec $CONTAINER mysql -u$DB_USER -p$DB_PASS -e "CREATE DATABASE \`$DB_NAME\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;" >/dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✅ 성공"
else
    echo "❌ 실패"
    exit 1
fi

echo ""
echo "🚀 Import 시작..."

docker exec -i $CONTAINER mysql -u$DB_USER -p$DB_PASS $DB_NAME < "$TMP_SQL" >"$LOG_FILE" 2>&1

if [ $? -eq 0 ]; then
    echo "🎉 Import 성공"
else
    echo "❌ Import 실패"
    echo "에러 로그:"
    tail -n 20 "$LOG_FILE"
    exit 1
fi

echo ""
echo "🧹 임시파일 정리"
rm -f "$TMP_SQL"

echo "✅ 모든 작업 완료"
```

---

# Import Script 실행

실행 권한 부여

```bash
chmod +x scripts/import_db.sh
```

실행

```bash
./scripts/import_db.sh
```

---

# 개발환경 구성

```text
Host

├── Docker Compose
│
├── Nginx
│
├── PHP-FPM 7.2
│
├── MariaDB 10.6
│
└── Source Code (Volume Mount)
```

개발자는 Host에서 소스를 수정하면 Volume Mount를 통해 컨테이너 내부에 즉시 반영되며, Import Script를 통해 운영 Dump를 빠르게 개발환경으로 복원할 수 있습니다.