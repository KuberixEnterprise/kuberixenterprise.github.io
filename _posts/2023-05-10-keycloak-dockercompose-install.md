---
layout: post
title: "Docker Compose를 사용한 Keycloak 설치"
description: docker compose를 사용하여 keycloak 21.1.1버전 설치
tags:
- keycloak
- 21.1.1
- docker compose
---

keycloak 설치를 위해 [Getting started](https://www.keycloak.org/guides#getting-started){:target="_blank"} 문서를 찾아보면 docker, kubernetes, openjdk, openshift, podman 5가지 방법이 소개되어 있습니다. 이중 docker를 사용한 문서를 확인해보면 빠른 시작을 위해 dev모드 기준으로 설명되어 있으며, docker compose를 사용한 시작 방법은 가이드되어 있지 않았습니다. 그래서 **docker compose**를 사용하여 현재 최신 버전인 **keycloak 21.1.1**을 기준 **production** 모드로 설치하는 문서를 작성하게 되었습니다.

## 구성환경
Public 환경에 구성된 AWS EC2에 Certbot을 통해 Let's Encrypt 인증서를 발급/갱신하고, Nginx에서 이 인증서를 사용하여 TLS Termination이 일어나며 proxy 설정을 통해 docker compose로 동작하는 서비스에 연결되는 구조입니다.
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/4e21447b-4af7-48ab-9b2f-5add4a8200b4)

## 설치 시 고려사항

### docker compose를 사용하려면?
keycloak 공식 문서에 docker compose를 사용하여 시작하는 가이드가 없었기 때문에 우선 [Running Keycloak in a container](https://www.keycloak.org/server/containers){:target="_blank"} 문서를 확인하였습니다. 크게 두 가지 방법으로 나뉘는데, 첫번째는 미리 빌드된 이미지를 사용하는 방법이며 두번째는 표준 시작 방법입니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/b3d588f6-9d04-45ca-b555-cd7499c436bb)
*<center>미리 빌드된 이미지 사용</center>*

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/c6d134f8-77de-475a-8078-2cb4e100aae8)
*<center>표준 시작</center>*

첫번째인 미리 빌드된 이미지를 사용하는 방법은 컨테이너가 다시 프로비저닝 될 가능성이 있어 빠르게 시작되어야 하는 **쿠버네티스**와 같은 클러스터 환경에 적합해 보입니다. 그래서 두번째 방법인 표준 시작 방법에 **docker**로 시작하는 커맨드를 참고하여 docker-compose.yml 파일을 작성하였습니다.

### Keycloak Wildfly와 Quarkus 배포판
keycloak 16버전까지는 **Wildfly** 배포판이 사용되었지만 20버전부터는 **Quarkus** 배포판만 지원하고 있으며 주요 변경 사항은 아래와 같습니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/6cdfc419-2b10-4105-bfae-9dc520d23b80)
*<center>주요 변경 사항</center>*

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/48e2f1cb-cd86-4f5a-8d10-312727ddaff4)*<center>keycloak 20버전부터 wildfly 배포판 지원 종료</center>*

배포판 변경이 일어나면서 기존 환경 변수 이름이나 사용방법이 변경되었기 때문에, 기존 설정을 이해한 후 공식 가이드 문서를 참고하여 마이그레이션 하기를 추천하고 있습니다. 저는 새로운 환경에 21.1.1 버전으로 설치하는 것이기 때문에 [config](https://www.keycloak.org/server/all-config){:target="_blank"} 가이드를 참조하여 환경변수를 설정하였습니다.

### 로드밸런서 혹은 리버스 프록시 사용 시
#### 프록시 모드 선택
Nginx를 기점으로 **TLS Termination** 되는 환경이고, 내부 통신은 암호화할 필요가 없었기 때문에 Nginx와 Keycloak간에 HTTP 통신을 사용하기 위한 **edge** 모드로 설정했습니다.
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/8e1b5cc0-e19f-4084-b9c0-ad3dda0d09db)
*<center>프록시 모드 종류</center>*

#### 리버스 프록시 헤더 설정
proxy를 거치면서 헤더가 재정의되는 경우 **일부 keycloak 기능**과 **administraion console** 접속이 정상적으로 동작하지 않을 수 있다고 합니다. 그래서 nginx.conf 설정에서 __X-Forwarded-*__ 헤더에 대해 설정을 추가하였습니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/2f0f32af-e05e-4b38-9627-fe758c37f9ce)
*<center>reverse proxy 사용시 헤더 설정 요구사항</center>*

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/e4ac020b-0050-4f2f-8c3d-8afb5c48daaf)
*<center>Administration Console 접속 불가 증상. 로딩 창에서 넘어가질 않는다.</center>*

### 프로덕션 모드 사용
start 커맨드 사용 시 production 모드로 시작하게 되는데, **default 값이 true**로 설정되면서 체크하는 부분이 생기기 때문에 docker-compose.yml 파일의 KC_HOSTNAME 환경변수를 추가적으로 설정해주었습니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/f8b33191-8ce6-4a09-aebc-f886fc316aaf)
*<center>production 모드로 시작 시 hostname 설정 에러</center>*

## 설정
### nginx
```
server {
       server_name keycloak.example.com;
       location / {
                   proxy_pass http://localhost:8080;
                   proxy_redirect off;
                   # 일부 keycloak 기능 및 administration console 접속을 위한 헤더 설정
                   proxy_set_header HOST $host;
                   proxy_set_header X-Forwarded-For $remote_addr;
                   proxy_set_header X-Forwarded-Proto $scheme;
                   proxy_set_header X-Forwarded-Port $server_port;
       }
       listen 443 ssl;
       ssl_certificate  /etc/letsencrypt/live/keycloak.example.com/fullchain.pem;
       ssl_certificate_key  /etc/letsencrypt/live/keycloak.example.com/privkey.pem;
       include /etc/letsencrypt/options-ssl-nginx.conf;
       ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}
```
```
nginx -s reload
```

### docker compose
```
services:
  postgres:
    image: postgres
    volumes:
      - ./volume/postgresql/data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: password
      TZ: Asia/Seoul
  keycloak:
    image: quay.io/keycloak/keycloak:21.1.1
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: password
      TZ: Asia/Seoul
      # keycloak이 quarkus를 사용하는 버전부터는 기존 환경변수 이름이 KC_*로 변경된 것이 많음
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: password
      KC_HOSTNAME: keycloak.example.com # production 모드 사용 시 정의
      KC_PROXY: edge # edge proxy mode 사용
    command:
      - start # production 모드로 시작
    ports:
      - 8080:8080
    depends_on:
      - postgres
```
```
docker compose up -d
```

## 마치며

현재 keycloak 설치를 위한 옵션이나 환경변수들을 구글링 해 보면 옛날 버전인 Wildfly 기준으로 설명되어 있는 글이 많습니다. 환경 변수 이름 및 사용방법이 변경된 부분이 있기 때문에, 현재 사용하는 keycloak 배포판에 맞춰 검색하거나 공식 문서를 찾아보는게 좋을 것 같습니다.

## 참고
* [Guides - Keycloak](https://www.keycloak.org/guides#getting-started)
* [Running Keycloak in a container - Keycloak](https://www.keycloak.org/server/containers)
* [Migrating to Quarkus distribution - Keycloak](https://www.keycloak.org/migration/migrating-to-quarkus)
* [Using a reverse proxy - Keycloak](https://www.keycloak.org/server/reverseproxy#_proxy_modes)
* [Configuring Keycloak for production - Keycloak](https://www.keycloak.org/server/configuration-production)
* [Proxy 환경에서 client IP 얻기 - tistory.com](https://cornswrold.tistory.com/442)