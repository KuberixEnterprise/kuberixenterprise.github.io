---
layout: post
title: "EKS의 IP 할당 이슈"
description: Node에서 사용 가능한 IP 부족 문제
tags:
- EKS
- IP 부족
- troubleshooting
author: 이화경
---

지난 달, 현재 운영하고 있는 고객사의 서비스가 DevOps 팀의 EKS로 넘어오는 작업이 있었습니다.

해당 작업 진행 후 EKS IP가 부족하여 Pod가 제대로 뜨지 못하는 이슈가 발생했고, 
재발 방지와 동일한 이슈 발생시 빠른 해결을 위해 이 문제의 원인과 해결 방안에 대해 공유하는 글을 작성하게 되었습니다.


## 문제의 배포방식과 리소스 설정

문제의 원인 설명에 앞서서 보다 쉬운 납득과이해(?)를 위해 문제가 발생한 환경의 배포 방식과 설정된 리소스에 대해 간단하게 공유하겠습니다. ^!^

우선, 이슈가 생긴 환경은 Blue/Green 배포 방식을 사용하고 있습니다.

Blue/Green 배포 방식은 무중단 배포 방식으로 새로운 배포가 생기면 먼저 배포되어 있던 구버전(Blue)과 신버전(Green)을 바꿔치기 하는 것입니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/32283544/d0ac5038-7549-49c0-bfd7-775412f2374e)


바꿔치기 하기 전 구버전에 떠있던 Pod 개수만큼 신버전의 Pod개수를 늘리게 되고, 이 때 Pod는 평상 시보다 2배가 됩니다..!!
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/32283544/9d23f28b-9976-4923-933c-553ab66ec557)

**그렇습니다. 문제는 이 구간에서 발생했습니다.**

*(이후 과정이 궁금하신 분은 ([Blue/Green 배포](https://devlog-wjdrbs96.tistory.com/300))를 참고해주세요.)*


그리고 급박하게 전환 일정이 잡히면서 필요한 Pod의 리소스를 정확히 파악 하지 못하고 대략적으로 기존 서비스의 EC2와 동일하게 Pod 리소스를 산정했습니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/32283544/08332a21-98bf-4256-aff3-f559822f24c4)

*<center>이것도 이슈 발생에 한 몫..</center>*

Pod가 위와 같은 스펙을 가지게 되면서 Node 1개에 Pod 1개가 뜨게 되었습니다..



## Trouble Shooting

### 이슈
서비스를 EKS로 옮기고 나서 모니터링했을 때까지만 해도 순조로웠습니다..
![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/32283544/41e2584a-ed20-4041-972a-c5af60941446)
*<center>평온한 초록색 하트들</center>*

하지만 ~~*위에서 스포한대로*~~ Blue/Green 스위치가 동작되면서 문제가 생겼습니다..!!..

스위칭되는 과정에서 Green 환경의 Pod가 Blue 환경의 Pod 개수만큼 늘어났고, Pod의 스펙이 커서 Node도 그만큼 늘어나게 되었습니다.

이 때 Node Group에서 사용할 수 있는 IP가 빠르게 사라졌고(..) 그로 인해 몇몇 Pod가 IP를 할당받지 못해 정상적으로 실행되지 못하는 현상이 발생했습니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/32283544/af069fa6-cd59-494b-baea-b056c5d220ad)
<center>*지금은 여유롭지만 당시에는 모두 0이었습니다..*</center>



### 해결

Pod를 제대로 띄우기 위해서는 IP 확보를 해야 했습니다.

당장 VPC를 새로 생성할 수는 없어서 각 Node 내 IP를 많이 쓰고 당장 사용하지 않고 있는 Daemonset인 Prometheus를 삭제했습니다.

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/32283544/f28809f7-b46d-4585-9f56-72be678437cc)

그런데 삭제하고 1시간이 지나도 IP를 반납하지 않았습니다... ~~제발IP돌려주세요~~

Prometheus을 삭제했음에도 불구하고 IP가 돌아오지 않은 이유를 찾아보던 중 한 [블로그글](https://aws-diary.tistory.com/146)을 발견했습니다.

해당 글에서 aws-node Daemonset 세팅과 관련하여 MINIMUM_IP_TARGET과 WARM_IP_TARGET에 대해 알게 되었습니다.

요약해서 설명하면 **WARM_IP_TARGET**은 Pod의 Secondary IP 값이고, **MINIMUM_IP_TARGET**은 ENI에서 IP를 미리 선점해가는 개수입니다.
해당 값들은 아래 명령어를 통해 확인이 가능합니다.

 ```
 kubectl describe daemonset -n kube-system aws-node
 ```

![image](https://github.com/KuberixEnterprise/kuberixenterprise.github.io/assets/32283544/394e8c84-8533-4508-ae76-c36722ae060b)
<center>참고 이미지</center>


각 Node가 고정적으로 필요한 IP 개수는 대략 6개였는데 위의 명령어로 확인해보니 **MINIMUM_IP_TARGET**이 13으로 설정되어 있었고, 사용하고 있는 IP 개수보다 더 많이 IP를 선점하고 있어서 Prometheus를 삭제해도 IP를 반납하지 않았던 것이었습니다.

그래서 아래 명령어로 **MINIMUM_IP_TARGET**을 8개로 변경하였더니 빠르게 IP를 뱉어냈고 확보한 IP로 Pod가 정상적으로 생성되면서 Blue/Green 스위치가 성공적으로 이루어졌습니다..!

```
kubectl set env ds aws-node -n kube-system MINIMUM_IP_TARGET=8
```



## 참고
* [Blue/Green 배포 방식](https://devlog-wjdrbs96.tistory.com/300)
* [EKS - Secondary IP](https://aws-diary.tistory.com/146)
