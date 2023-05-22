---
layout: post
title: "Keycloak Google 로그인 연동"
description: Keycloak Identity Provider를 사용한 Google 로그인 연동 방법
tags:
- keycloak
- google
- identity provider
author: 박현성
---

쿠버릭스는 기본적으로 Google Workspace 계정을 발급받아 Jira, Confluence, Slack과 같은 협업 도구도 Google 로그인을 통해 사용하고 있습니다. 이 문서에서는 Gitlab, Harbor와 같은 직접 설치한 툴도 keycloak을 거쳐 Google 계정을 통해 로그인 할 수 있도록, [Integrating identity providers 문서](https://www.keycloak.org/docs/latest/server_admin/#_google){:target="_blank"}를 참고하여 연동하는 방법에 대해 다룹니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/8fdadd0d-e8e2-4e76-a92f-363995fbebe2)
*<center>Identity Broker flow</center>*

## 구성환경
* 조직에 속한 Google Workspace 계정
* [keycloak 21.1.1](https://blog.kuberix.co.kr/2023/05/10/keycloak-dockercompose-install.html){:target="_blank"}
* [harbor 2.7.0](https://blog.kuberix.co.kr/2023/05/17/keycloak-harbor.html){:target="_blank"}

## Keycloak 리디렉션 URI 확인
google OAuth 설정에 입력 할 리디렉션 URI를 위해, 미리 keycloak에서 identity provider를 생성 하여 복사해둡니다.
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/947cb7e4-aa7e-4dbc-a98b-52043689886b)
*<center>Redirect URI 복사</center>*

## Google OAuth Client 설정
Google Cloud의 **OAuth Client** 및 **OAuth 동의 화면** 설정을 통해 조직 내 사용자가 로그인 할 수 있는 클라이언트를 생성합니다.

### OAuth 클라이언트 ID
* 애플리케이션 유형 : 웹 애플리케이션
* 이름 : keycloak
* 승인된 리디렉션 URI : https://keycloak.example.com/realms/test/broker/google/endpoint

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/27389f4c-204c-4364-95d7-ed3e2cd85596)
*<center>생성 시 나오는 클라이언트 ID와 클라이언트 보안 비밀번호 복사</center>*

### OAuth 동의 화면
* 앱 이름 : keycloak
* 사용자 유형 : 내부
* 사용자 지원 이메일 : admin@example.com
* 승인된 도메인 : example.com
* 개발자 연락처 정보 : admin@example.com

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/bc1de2b1-b09c-4164-82d5-41382c781699)
*<center>사용자 유형을 내부로 설정 시, 조직에 속한 사용자만 로그인 제한</center>*

## Keycloak Identity Providers 설정
OAuth 클라이언트 ID 생성 시 만들어지는 클라이언트 ID와 클라이언트 보안 비밀번호를 입력 후 저장합니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/c20427c7-487b-4504-9bfc-5750c76a696e)
*<center>클라이언트 ID, 클라이언트 보안 비밀번호 입력 후 저장</center>*

## 테스트
### Google 로그인 화면
미리 연동해둔 harbor를 통해 keycloak 로그인 페이지를 보면 아래 Google 로그인 버튼이 생긴 것을 확인할 수 있습니다.
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/971a065d-3b9c-46c5-88fe-b3be3d0c3986)

### 조직 외 사용자 로그인
OAuth 동의 화면에서 사용자 유형을 내부로 선택했기 때문에, 같은 조직에 속한 사용자만 로그인 가능한 것을 볼 수 있습니다.
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/92906503/54345230-80fa-4328-af2a-d5d2226293ae)
*<center>조직 외 사용자 엑세스 차단</center>*

## 마치며
처음엔 Keycloak에서 사용자를 직접 추가 및 관리하려고 했으나, 입사자 및 퇴사자 Google 계정을 이미 타 부서에서 관리하고 있는 상황이였습니다. 그리고 Google 계정 로그인을 통해 기존 사용하던 협업 도구들이 있었기 때문에, Google 계정만 로그인되어 있으면 사내 솔루션에 접근할 수 있도록 구성하는 방향으로 변경하게 되었습니다.  
결과적으로 자체 Keycloak 사용자 관리 시 연동하려고 했던 Atlassian Access 유료플랜을 사용하지 않아도 되게 되었고, 계정관리도 타 부서에서 관리하며, Google 계정 로그인 시 인증도 자동으로 거치게 되는 구성이 되었습니다.

## 참고
* [Server Administration Guide (Integrating identity providers)](https://www.keycloak.org/docs/latest/server_admin/#_google){:target="_blank"}