---
layout: post
title: "Gitlab version upgrade"
description: Gitlab version upgrade와 후에 생긴 이슈 해결에 대한 내용
tags:
- blog
---

현재 아모레는 GitLab CE 15.2 버전으로 설치되어 있으며, 해당 버전은 원격코드 실행 취약점(CVE-2022-2992)이 존재하여 버전 업을 하게 되었습니다.


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


## Troubleshooting
### 이슈
무사히 잘 gitlab version 업을 하고 테스트 파이프라인을 돌렸더니 기존에 잘 사용하고 있던 파이프라인이 오류가 났습니다..
오류가 나는 부분을 살펴보니 variable를 제대로 읽어오지 못해 artifacts/paths가 정상작동(?) 하지 않았고, 
artifact에 빌드된 파일이 제대로 올라가지 못하니까 해당 폴더에서 파일을 읽어오는 stage에서 문제가 생긴 것이었습니다.

### 해결
구글링 중 Gitlab에서 현 상황과 똑같은 문제로 이슈를 생성한 글을 보았습니다.
해당 글에서 Gitlab 버전 변경 시 helper image도 같이 변경해야 한다는 글을 발견했습니다.


인터넷에 서치해보니, gitlab version 변경 시 helper image도 같이 변경해주어야 한다는 글을 보았습니다.
helper image를 변경하는 데에는 여러가지 이유가 있는데, Gitlab Docs에서
1. job 실행 속도 높이기 위함
2. 보안 문제
3. 빌드 환경이 폐쇄망
4. 소프트웨어 추가
이렇게 4가지를 이유로 들었습니다.

그 중 2번에 해당 된다고 생각되어 helper image를 변경하였더니 파이프라인이 정상작동했습니다.

참고 url: 
- https://gitlab.com/gitlab-org/gitlab/-/issues/388948
- https://docs.gitlab.com/runner/configuration/advanced-configuration.html#override-the-helper-image
