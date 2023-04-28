---
layout: post
title: "GitLab version upgrade"
description: GitLab version upgrade와 후에 생긴 이슈 해결에 대한 내용
tags:
- blog
---


2022년 8월 말, GitLab에서 보안 취약점에 대한 업데이트를 발표했습니다.
해당 발표는 GitLab의 GitHub API 엔드포인트에서 가져오기 기능을 이용해 발생하는 원격코드 실행 취약점 ([CVE-2022-2992](https://about.gitlab.com/releases/2022/08/30/critical-security-release-gitlab-15-3-2-released/)) 등 15개를 조치했다는 내용입니다.

영향 받는 버전은
- GitLab CE/EE 11.10 이상  15.1.6 미만 버전
- GitLab CE/EE 15.2 이상 15.2.4 미만 버전
- GitLab CE/EE 15.3이상 15.3.2 미만 버전
이고 고객사에서 사용하는 GitLab 버전(15.2)은 GitLab CE 15.2입니다.

그래서 현재 운영하고 있는 고객사의 ISMS 인증 대응하는 과정에서 위의 내용인 원격코드 실행 취약점(CVE-2022-2992)이 발견되었고 그로 인해 GitLab 버전 업을 진행하게 되었습니다.

이 글은 GitLab 버전 업 후 생겼던 이슈에 대해 공유하고자 글을 작성하게 되었습니다.
GitLab CI를 사용하고 GitLab 버전 업 계획이 있는 사람들에게 도움이 되길 바랍니다.. ^_^


# GitLab version upgrade 작업 항목
- GitLab & GitLab Runner  볼륨 백업
- GitLab & GitLab Runner  15.2 버전을 15.10 버전으로 업그레이드

## 작업 상세
1. GitLab & GitLab Runner 도커볼륨 백업
```
# gitlab backup
cd /data
sudo cp -r gitlab/ gitlab.bak

# gitlab runner backup
cd /data
sudo cp -r gitlab-runner/ gitlab-runner.bak
```

2. GitLab 대시보드에서 기존 GitLab & Runner 버전 확인
GitLab : 15.2.2
GitLab Runner 1,2,3,4 : 15.2.2

3. GitLab 업그레이드 버전 이미지 받기
```
docker pull gitlab/gitlab-ce:15.10.2-ce.0
# 이미지 확인
docker images -a
```

4. docker-compose.yml 파일 수정
```
cd /data/containers/
sudo vi docker-compose.yml

# docker-compose.yml
...
version: "3.6"
services:
  gitlab:
    image: gitlab/gitlab-ce:15.10.2-ce.0
...
```

5. GitLab 재시작
```
# docker-compose 실행 명령어
docker-compose up -d
```

6. GitLab Runner도 위와 동일하게 진행



## Troubleshooting
### 이슈
무사히 잘 GitLab version 업을 하고 테스트 파이프라인을 돌렸더니 기존에 잘 사용하고 있던 파이프라인이 오류가 났습니다..
오류가 나는 부분을 살펴보니 variable를 제대로 읽어오지 못해 artifacts/paths가 정상작동(?) 하지 않았고, 
artifact에 빌드된 파일이 제대로 올라가지 못하니까 해당 폴더에서 파일을 읽어오는 stage에서 문제가 생긴 것이었습니다.
![image](https://user-images.githubusercontent.com/32283544/234451235-d5614865-061a-46db-bb3c-ccec66c27523.png)



### 해결
해결방안을 찾던 중 [GitLab 공식 프로젝트(?)의 Issue](https://gitlab.com/gitlab-org/gitlab/-/issues/388948)에서 gitlab version 변경 시 helper image도 같이 변경해주어야 한다는 글을 보았습니다.
helper image를 변경하는 데에는 여러가지 이유가 있는데, [GitLab Docs](https://docs.gitlab.com/runner/configuration/advanced-configuration.html#override-the-helper-image)에서
1. job 실행 속도 높이기 위함
2. 보안 문제
3. 빌드 환경이 폐쇄망
4. 소프트웨어 추가
이렇게 4가지를 이유로 들었습니다.

현재 고객사는 폐쇄망으로 구축되어 helper image를 ECR에 올리고 그걸 가져와서 사용하는 형식입니다.
그래서 위의 이유 중 3번에 해당 한다고 생각되었고, helper image를 변경하였더니 파이프라인이 정상작동했습니다.

ECR에 helper image의 태그 목록입니다.

![image](https://user-images.githubusercontent.com/32283544/235035995-5129395d-4ae4-4e0f-8d54-a977383bac35.png)

helper image는 각 runner의 /etc/gitlab-runner/config.toml에서 수정 가능합니다.
![image](https://user-images.githubusercontent.com/32283544/235035507-e321f7a4-47d6-4fa4-bbf9-8c7d7f2209d6.png)



## 복구 절차
이 파트는 되도록 따라하는 일이 안생기는 것이 좋겠지만, 혹시나 버전 업 후 문제가 생길 경우를 대비하여 작업하기 전 상태로 돌리는 방법에 대해서 기술했습니다. :)

예상가능한 문제로
1. 이미지에 대한 문제
2. Data에 대한 문제
2가지 경우로 생각됩니다.

복구 과정을 경우에 따라 나눠서 작성했습니다.

1. 이미지에 대한 문제
다운로드 과정에서 네트워크 환경이 좋지 못했다던가, 이미지가 이상(?)하다던가, 등 버전 업 이미지에 문제가 있는 경우입니다.

docker-compose.yml파일에서 이전에 사용하던 이미지로 수정한 뒤, GitLab 재시작
```
# docker-compose.yml 파일에서 기존 이미지로 수정 후, Gitalb 재실행

cd /data/containers/
sudo vi docker-compose.yml
docker-compose up -d
```

2. Data에 대한 문제
이미지 변경하는 과정에서 데이터가 날아갔다던가(...), 유실이 생겼다던가 등 데이터에 문제가 있는 경우입니다.

백업해두었던 볼륨 폴더를 도커 볼륨으로 사용하도록 수정한 뒤, GitLab 재시작
```
# docker-compose.yml
...
services:
  gitlab:
    image: gitlab/gitlab-ce:15.10.2-ce.0
  ...
    volumes:
      - /data/gitlab.bak/datak:/var/opt/gitlab
      - /data/gitlab.bak/logs:/var/log/gitlab
      - /data/gitlab.bak/config:/etc/gitlab
...
```
