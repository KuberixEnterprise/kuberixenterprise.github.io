---
layout: post
title: "Gitlab version upgrade"
description: Gitlab version upgrade와 후에 생긴 이슈 해결에 대한 내용
tags:
- blog
---

현재 아모레는 GitLab CE 15.2 버전으로 설치되어 있으며, 해당 버전은 원격코드 실행 취약점(CVE-2022-2992)이 존재하여 버전 업을 하게 되었다.



# Gitlab version upgrade 작업 항목
- Gitlab & Gitlab Runner  볼륨 백업
- Gitlab & Gitlab Runner  15.2 버전을 15.10 버전으로 업그레이드

## 작업 상세
1. Gitlab & Gitlab Runner 도커볼륨 백업
```
# gitlab backup
cd /data
sudo cp -r gitlab/ gitlab.bak

# gitlab runner backup
cd /data
sudo cp -r gitlab-runner/ gitlab-runner.bak
```

2. Gitlab 대시보드에서 기존 GitLab & Runner 버전 확인
Gitlab : 15.2.2
Gitlab Runner 1,2,3,4 : 15.2.2

3. Gitlab 업그레이드 버전 이미지 받기
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

5. Gitlab 재시작
```
# docker-compose 실행 명령어
docker-compose up -d
```

6. Gitlab Runner도 위와 동일하게 진행


## 복구 절차
버전 업한 Gitlab에 문제가 생겼을 때 이전으로 복구하는 방법

1. 이미지에 대한 문제가 있을 시
docker-compose.yml파일에서 이전에 사용하던 이미지로 수정한 뒤, Gitlab 재시작
```
# docker-compose.yml 파일에서 기존 이미지로 수정 후, Gitalb 재실행

cd /data/containers/
sudo vi docker-compose.yml
docker-compose up -d
```

2. Data에 대한 문제가 있을 시
백업해두었던 볼륨 폴더를 도커 볼륨으로 사용하도록 수정한 뒤, Gitlab 재시작
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