---
layout: post
title: "[바로가기] Blue-green 무중단 배포"
category: Side Projects
---

## Table of contents

- [1. Blue-Green 배포란?](#1-blue-green-배포란)
- [2. Blue-Green 배포 과정](#2-blue-green-배포-과정)
  - [2.1 Nginx Upstream 설정](#21-nginx-upstream-설정)
  - [2.2 무중단 전환 스크립트 작성](#22-무중단-전환-스크립트-작성)
  - [2.3 Docker Compose 설정](#23-docker-compose-설정)
- [3. 배포 프로세스 정리](#3-배포-프로세스-정리)

&nbsp;  
&nbsp;

사이드 프로젝트로 AI 장소 추천 어플리케이션, '바로가기' 개발을 진행했습니다.
이번 프로젝트에서 서버 세팅을 할 때 제 목표는 크게 2가지였습니다.

- 무중단 배포 (Blue-Green)
- 모니터링 (Prometheous, Grafana)

그중 첫 번째인 Blue-Green 배포에 대해 기록해 보려고 합니다.

<br>

## 1. Blue-Green 배포란?

Blue-Green 배포는 두 개의 동일한 운영 환경(Blue, Green)을 구성하고, 배포 시 트래픽을 순간적으로 전환하는 무중단 배포 방식입니다.

현재 서비스 운영 중인 환경(Blue)과 서비스 요청을 받지 않는 환경(Green)이 있습니다. 코드 수정 후 Green 환경에 새 버전을 배포하고, 테스트 완료 후 트래픽을 Green으로 전환합니다.

롤링 배포, 카나리 배포 등 다양한 배포 방법이 있었지만 Blue-Green 배포를 선택한 이유는 2가지입니다.

<br>

- 서비스의 크기: 아직 출시 전 서비스이고, 규모가 크지 않기 때문에 컨테이너 2개를 사용하는 블루 그린 배포 방식이 적합하다고 판단했습니다.
- 롤백 용이: 만약 새 버전에 이상이 생긴 경우, 즉시 이전 버전으로 롤백할 수 있다는 장점이 있기 때문에 선택했습니다.

<br>

## 2. Blue-Green 배포 과정

### 2.1 Nginx Upstream 설정

Nginx가 두 개의 백엔드 서버(8081, 8082 포트)를 인식하도록 upstream 설정을 추가했습니다.

```bash
upstream backend {
    server 127.0.0.1:8081;  # Green (현재 active)
}
```

upstream은 nginx가 대신 트래픽을 나눠 보내줄 백엔드 서버 묶음입니다. 즉, 여러 개의 백엔드 서버를 하나의 논리적인 그룹으로 묶는 것입니다. 이렇게 하면 nginx는 backend 서버가 여러 개라고 인식합니다.

만약 upstream 설정을 하지 않는 경우, 트래픽 대상은 딱 하나로 고정됩니다. 포트를 바꾸면 nginx 설정을 전체 수정해야 하고, 배포할 때 무조건 서비스가 끊기는 문제가 발생합니다. 즉, Blue-Green 배포가 불가능해집니다.

```bash
proxy_pass http://backend;
```

nginx로 들어오는 요청을 backend라는 upstream 그룹의 서버로 보내도록 하는 설정입니다.

<br>

### 2.2 무중단 전환 스크립트 작성

다음으로 트래픽을 변경하기 위한 무중단 전환 스크립트(switch.sh) 파일을 작성했습니다.

```bash
#!/bin/bash

NGINX_CONF="/etc/nginx/sites-available/example.conf"

if grep -q "8081" $NGINX_CONF; then
    echo "Switching: green → blue"
    sed -i 's/8081/8082/' $NGINX_CONF
else
    echo "Switching: blue → green"
    sed -i 's/8082/8081/' $NGINX_CONF
fi

nginx -t && systemctl reload nginx
```

무중단 전환 스크립트의 동작 방식은 아래와 같습니다.

- 현재 활성화된 포트를 확인합니다 (8081 or 8082)
- 반대 포트로 변경합니다
- Nginx 설정 검증 후 리로드합니다

<br>

### 2.3 Docker Compose 설정

두 개의 컨테이너를 각각 다른 포트로 실행하도록 설정했습니다.

```yaml
services:
  app-green:
    ports:
      - "8081:8080"

  app-blue:
    ports:
      - "8082:8080"
```

<br>

## 3. 배포 프로세스 정리

- 새 버전을 대기 중인 컨테이너에 배포합니다. (Green이 활성화 상태라면 Blue에 배포)
- switch.sh 스크립트를 실행해 트래픽을 전환합니다.
- 새 버전이 정상적으로 실행되는지 확인합니다. 이상이 생기면 즉시 롤백합니다.

<br>
