---
layout: post
title: "Keycloak Harbor 연동"
description: Keycloak OIDC Provider와 Harbor 로그인 연동 방법
tags:
- keycloak
- harbor
- oidc
---

harbor Authentication 설정을 통해 Keycloak OIDC Client와 연동하는 방법을 설명합니다. [Configure OIDC Provider Authentication](https://goharbor.io/docs/1.10/administration/configure-authentication/oidc-auth/) 문서와 [Keycloak as OIDC Provider for Harbor](https://medium.com/@panda1100/keycloak-as-oidc-provider-for-harbor-c25906481619) 블로그를 참고하였습니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/b25b351e-fc38-4961-8faf-6160d85c728b)
*<center>Keycloak OIDC Flow</center>*

## 구성환경
* [keycloak 21.1.1](https://blog.kuberix.co.kr/2023/05/10/keycloak-dockercompose-install.html)
* harbor 2.7.0

## 설치 전 확인
* harbor 설치 시 생성되는 **admin**(관리자) 계정 외 **사용자 계정이 없는 상태**에서만 설정이 가능합니다.
* OIDC Provider 연동시 harbor에서 **사용자를 추가할 수 없습니다.**

## Harbor Redirect URI 확인
Harbor admin 계정 로그인 > Administration > Configuration > Authentication
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/502d5c5e-7c45-4be2-9fab-bfdbecfbc599)
*<center>Keycloak Client 설정에서 사용할 Redirect URI 복사</center>*

## Keycloak 설정
### OIDC Client 생성
Keycloak > Administration Console > (Realm) > Clients > Create Client
* Client type : OpenID Connect
* Client ID : harbor
* Client authentication : On
* Valid redirect URIs : ${**Redirect URI**}

### Client Secret 복사
Keycloak > Administration Console > (Realm) > Clients > harbor > Credentials
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/71401d07-5998-4cdf-86fe-b642e2d8e381)
*<center>Harbor Authentication에 설정할 Client Secret 복사</center>*

### OIDC Endpoint 복사
Keycloak > Administration Console > (Realm) > Realm settings > Endpoints > 
OpenID Endpoint Configuration
```
{"issuer":"https://keycloak.example.com/realms/test"}
```
*<center>Harbor Authentication에 설정할 OIDC Endpoint 복사</center>*

## Harbor 설정
Harbor admin 계정 로그인 > Administration > Configuration > Authentication
* Auth Mode : OIDC
* OIDC Provider Name : keycloak
* OIDC Endpoint : ${**OIDC Endpoint**}
* OIDC Client ID : harbor
* OIDC Client Secret : ${**Client Secret**}
* OIDC Scope : openid,profile,email,offline_access  

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/fbaca181-6a96-4bf0-b5f8-48a1794dfea7)
*<center>Harbor Authentication 설정</center>*

## 테스트

### 사용자 추가
Keycloak > Administration Console > (Realm) > Users > Add user
* Username : username
* Email : username@example.com

### 패스워드 설정
Keycloak > Administration Console > (Realm) > Users > username > Credentials > Set password
* Password : password
* Password confirmation : password

### Harbor 로그인
harbor > LOGIN VIA OIDC PROVIDER
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/179e8db9-3ce1-4576-aa9c-820646d79bb8)
*<center>위는 Keycloak 사용자 로그인, 아래는 관리자 로그인</center>*

### CLI secret 확인
harbor 로그인 후 우측 상단 유저이름의 User Profile 메뉴를 보면 CLI secret이 별도로 존재하는 것을 볼 수 있습니다. Docker나 Helm CLI 사용 시 필요한 key입니다.
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/5ddb7bc4-f2ea-4c52-bb3c-764325f8b4df)
*<center>Docker나 Helm 로그인 시 Username은 ID, CLI Secret은 Password로 사용</center>*

### Docker 로그인
```
docker login harbor.example.com
Username : [Username]
Password : [CLI secret]
```

## 마치며
참고한 블로그의 keycloak 버전과 설치한 keycloak 버전 차이가 있어 설정 화면이 조금 달랐지만 대부분 그대로 설정 가능했고, harbor에서 OIDC Provider를 통한 인증 설정 화면을 제공하기 때문에 쉽게 연동이 가능했습니다.  
그리고 keycloak 사용자로 harbor에 로그인하게 되면 keycloak 사용자 정보를 바탕으로 harbor에도 사용자가 생성되는데, admin 계정으로 로그인하여 프로젝트 접근 사용자 추가 및 역할 설정으로 권한을 관리할 수 있었습니다.

## 참고
* [Configure OIDC Provider Authentication](https://goharbor.io/docs/1.10/administration/configure-authentication/oidc-auth/)
* [Keycloak as OIDC Provider for Harbor](https://medium.com/@panda1100/keycloak-as-oidc-provider-for-harbor-c25906481619)
* [A Quick Guide To Using Keycloak For Identity And Access Management](https://www.comakeit.com/blog/quick-guide-using-keycloak-identity-access-management/)