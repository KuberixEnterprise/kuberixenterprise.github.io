---
layout: post
title: "데이터 웨어하우스와 BPMN 을 기반한 이커머스 업무자동화 시스템 개발기"
description: 데이터 웨어하우스와 BPMN 을 기반한 이커머스 업무자동화 시스템 개발기
categories:
- aws
date: 2021-04-16 11:00:00
tags:
- redshift
- bpmn
- aws
---

# 데이터 웨어하우스와 BPMN 을 기반한 이커머스 업무자동화 시스템 개발기

> 아래의 내용은 실제 프로젝트 도메인과 연관성이 없는 각색한 내용입니다.

기업 측면에서의 수요의 예측은 생산, 자재 및 물류 등의 계획과 관리 측면에서 중요합니다.
수요예측을 데이터를 기반하여 합리적으로 판단함으로써, 판매 목표설정, 설비투자 및 생산계획, 재고 조절,공급체인 관리, 마케팅전략 수립등에 중요한 데이터로 쓰일 수 있으므로 수요 예측은 기업의 SCM(Supplier Chain Management) 에서 가장 중요한 부분을 차지합니다.

최근 진행하고 있는 이커머스 프로젝트는, 사람이 수행하던 SCM 의 활동들인 분석,예측,발주,정산 등이 데이터에 근거하여 자동으로 수행되어 사람은 더 가치있는 일에 집중할 수 있도록 하는 것에 목적이 있고, 그 첫 시작이 수요예측 업무 자동화 였습니다.

이 글에서는 프로젝트를 진행하면서 선택한 아키텍처, 개발과정, 앞으로의 개선점을 적어보도록 하겠습니다.

# 아키텍처 선정과정

## 빅데이터와 BPMN 엔진

이 프로젝트가 다른 빅데이터 기반 웨어하우스 구축등의 프로젝트와 다른점은, 분석의 결과가 보고서 작성등의 일회성 분석요건에 그치지 않고 사업체의 비즈니스 프로세스와 밀접한 관련이 있어 하루에도 수차례씩, 업무의 필요에 의해 분석을 수행하고, 결과물을 다음 업무에 전달하여 또 다른 분석이 실행되는 업무의 연속성이 있다는 것 입니다. 그리고 분석에 필요한 데이터는 분석의 근간이 되는 기준데이터(매일 발생하는 입,출고 내역 등) 와 변수데이터(예측하고 싶은 날짜, 행사로 인한 예측 추가 판매 수량등) 로 나누어지는데, 변수데이터가 업무자의 필요에 의해 매 수행시마다 요건에 맞추어 변동이 된다는 점 입니다. 이러한 특징은 전통적인 데이터 웨어하우스를 제공하고 사전에 작성된 획일화된 쿼리를 매일 수행하는 방식으로는 성립될 수 없었습니다.

![](https://user-images.githubusercontent.com/13447690/114333176-a9b1fd00-9b82-11eb-977b-be403e231c32.png)

(프로젝트와 동일하지는 않지만 대략적으로) 수요예측에서 자동발주로 이어지는 프로세스의 흐름이 위의 그림과 같다고 생각 해 봅시다.

사람모양의 아이콘은 각 관계부서이고, 이 관계부서들은 각자의 권한 아래 그날그날의 데이터를 제공하거나 해당 날의 업무 특이사항으로 인해 변수데이터를 조작할 수 있습니다. 그리고 각 관계부서에서 수행한 분석결과는 다음 부서가 이어받아서 또 다른 분석을 수행합니다. **모든 부서가 제시간에 자기 할일을 오차없이 정직하게 수행한다면 정적인 워크플로우로 충분히 구현 가능한 상황이지만, 현실 업무는 그렇지 않습니다.**  모든 관련자가 작업을 변경하거나, 작업을 중단하거나 재게할 수 있습니다. 이렇게 변경이 가해진 산출물을 뒤이은 부서에서 다시 활용할 수 있도록 하고, (또는 정책에 따라 이전에 발생한 산출물은 사용할 수 없도록 막거나) 발생한 산출물의 흐름은 하나의 작업아이디로 묶어 관리할 수 있는 유연한 프로세스 관리가 필요 해 집니다. 

위의 특징을 아래와 같이 정리한 표 입니다.

| 특징  | 일회성 분석  | 비즈니스 프로세스를 위한 분석 |
|--------------------|----------------------|---------------|
| 수행 | 매일 특정 주기   |  필요할 때 즉시 |
| 수행처 | 데이터 팀 |  업무에 연관된 사람 |
| 사용처    | 일회성, 보고 또는 분석용  |  다음 업무에 사용 |
| 변수데이터처리  | 또다른 기준데이터를 변수로 활용 | 업무에 연관된 사람이 직접 주입가능 |
| 쿼리    | 정적   |  동적 |

---

위 요건을 맞추기 위해 활용한 것은 BPM 기반 프로세스 엔진입니다.
비즈니스 프로세스의 정의는 비즈니스 및 조직의 모델을 기반으로합니다. BPM 기반 엔진을 사용하여 프로세스 기반 애플리케이션을 생성하기 위해 프로세스를 자동화하는 개발자는 누가, 무엇을 작업하며 언제  실행해야하는지 고려하여 모든 권한을 가지고 설계를 진행할 수 있습니다.
BPM은 사용자가 미리 정의 된 순서로, 그리고 종종 미리 정의 된 시간 제한 또는 기한 내에 작업을 수행하고 데이터를 업데이트하도록 보장하는 것을 목표로합니다. 비즈니스 규칙은 프로세스 정의에 의해 시행됩니다.

프로젝트에 사용한 비즈니스 프로세스 엔진은 [https://www.activiti.org/userguide/#_introduction](https://www.activiti.org/userguide/#_introduction) 입니다. 비즈니스 프로세스 엔진은 모두 BPMN 2.0 스펙에 의거하여 동작하므로 얼마나 많은 기능을 지원하느냐보다 얼마나 오래되고 안정적인 프로젝트인가를 고려하여 선택하였습니다. Apache activiti 는 BPMN 엔진의 원조격이고 이로 인해 수많은 아류 프로젝트들이 생겨났습니다. 최근에는 Cloud 와 결합하고 다양한 데이터소스 인티그레이션 어탭터를 제공하는 데이터플로우 플랫폼들이 더 인기를 얻고 있어 Apache activiti 가 주목을 받지 못하고 있지만 다음의 이유 때문에 선택을 하였습니다.

1. 안정성
2. 최근까지도 커밋이 일어나고 있다.
3. 플랫폼으로 제공되는 형태가 아니라 java lib 으로 제공되므로 개발자가 자유롭게 커스터마이징 가능하다.
4. 요구사항만 수행하기에 기능이 충분하고 매우 가벼운 엔진이다.

다음은 BPMN 엔진의 구성요소와 어플레케이션에서 어떻게 동작하는지에 대한 설명입니다.

### 명세서

- 프로세스의 흐름을 기록한 BPMN 스펙 문서이다. 
- 실제로 모델링할때는 [https://bpmn.io/](https://bpmn.io/) 등의 사이트에서 그림을 그린후, export 하면 기본 xml 을 얻을 수 있다.
- xml 은 각 작업을 담당할 serviceTask 들과,  serviceTask 사이의 흐름을 연결하는 sequenceFlow 로 구성되어 있다.
- 다음은 [https://bpmn.io/](https://bpmn.io/) 에서 분석 작업 중 하나를 불러오기 했을때 나타나는 그림이다. 실제 하나의 분석작업에는 여러단계의 데이터처리 과정이 필요함을 볼 수 있다.
![](https://user-images.githubusercontent.com/13447690/114335971-e97be300-9b88-11eb-9e9d-47421cb0a5bf.png)

### 프로세스

- 워크플로우 엔진이 명세서를 사용하여 하나의 워크플로우를 실행하였을 경우, 이것을 프로세스 라고 한다.
- 프로세스는 명세서의 시퀀스플로우대로 serviceTask 들을 실행시키며,  이것의 상태를 관리한다.
- 프로세스는 시작시 변수를 주입할 수 있다. 그리고 serviceTask 들은 이 변수를 사용하여 매번 다른 동작을 할 수 있게 된다.
- 상태관리는 Activiti 데이터베이스 에 저장되며, 멀티 인스턴스 환경에서도 동시성 문제가 발생하지 않는다.

### 태스크

- 프로세스에 의해 실행되는 단계별 작업으로,  Bean 또는 Bean 이 아닌 일반 클래스가 수행된다.
- 태스크에서 워크플로우 변수를 가져와 자신만의 작업처리를 진행할 수 있다.
- 태스크에서 변수를 워크플로우에 주입하여 뒤이은 작업들이 변수를 활용하게 할 수 있다.

## 데이터 웨어하우스 선정

데이터 웨어하우스는 다음과 같은 이유로 AWS Redshift 을 선택하였습니다.

1. 운영에 영향을 미치지 않고 대용량 데이터 처리를 할 수 있는 콘텍스트가 필요했다.
2. 매 작업시 산출물이 10GB 로 많지 않아도 누적되면 부담이 되기에 스토리지 효율이 좋은 데이터베이스가 필요했다. S3 를 저장소로 활용할 수 있는 Redshift 스펙트럼과 Glue Catalog 는 정말 좋은 서비스였다.
3. Postgresql 을 기반한 SQL 문법이라, JDBC 드라이버를 통해 일반 RDS 처럼 접속하여 쓰면 되었다. 레드시프트를 사용하기 위해 특별한 준비과정이 필요없었다.
4. 생각보다 저렴한 가격과 만족했던 성능. 레드시프트 인스턴스 (2cpu, 16GB, 일비용 8000원) 하나로 충분히 진행 가능하였다. 업무 특성상 동시에 여러 프로세스가 잘 발생하지 않고, 프로세스가 20 ~ 30 분정도의 진행시간이 걸려도 용인될 수 있었다.
5. 매니지드 서비스만을 이용하여 3th party 솔루션 없이 구축 가능
6. Glue Catalog 를 통해 이미 AWS 에서 관리되고 있는 또다른 기준데이터 소스들과 쉽게 인티그레이션이 가능

![](https://user-images.githubusercontent.com/13447690/114467632-2990a380-9c25-11eb-94b7-66e97edb7c13.png)

레드시프트 선정이유 중 가장 마음에 드는 항목은 Glue Catalog 였습니다. Glue Catalog 는 다양한 소스에서 데이터 검색 및 추출, 데이터 강화, 정리, 정규화 및 결합이 가능하고 산출물을 여러 분산된 데이터 소스에 저장이 가능하였습니다.  해당 프로젝트를 수행했던 사업체의 데이터 플랫폼 팀에서는 이미 AWS EMR 을 사용하고 있었습니다. 이는 곧 AWS Glue Data Catalog 를 메타스토어로 활용하여 기존 사업체의 기준데이터들과 인티그레이션이 매우 용이한 구조입니다. 프로젝트가 진행될 수록 필요한 기준데이터 종류가 늘어나게 될 것인데, 별다른 노력없이 데이터 통합 ETL 이 가능하다는 점은 매우 강력한 장점이 될 것입니다.

# 개발과정

*개발과정에서 수립한 각 시스템의 개요도*
![](https://user-images.githubusercontent.com/13447690/114952419-2d286280-9e91-11eb-92ee-5855f6394d69.png)

개발과정에서 수립한 각 역할의 구조는 다음과 같습니다.

- S3
  - daily 폴더	
    - 매일 적재되는 기준데이터 저장 디렉토리
    - Redshift 의 스펙트럼 테이블 저장소로 파티션으로 나뉘어 있다.
  - job 폴더	
    - 프로세스 작업 단위마다 UUID 형태로 하나씩 생성
    - Redshift 의 스펙트럼 테이블 저장소로 파티션으로 나뉘어 있다.
  - excel 폴더
    - 각 프로세스 작업단계 마다 엑셀결과물 파일이 저장되는 곳이다.
  - search 폴더	
    - 프로세스가 아닌, 일회성 분석의 엑셀결과물 파일이 저장되는 곳이다.

- RedShift
  - Leader Node	
    - AWS 제공하는 레드시프트 JDBC 라이브러리를 사용하여 어플리케이션과 통신하며,  일반 RDBMS 처럼 DB커넥션을 맺고 sql 로 통신한다.
    - 쿼리요청에 대해 컴퓨트 노드들에 쿼리플랜을 할당한다.
    - 컴퓨트 노드에서 쿼리된 결과를 aggregate 한다
    - 전체 노드를 한대로 운용할 경우 리더노드가 컴퓨터노드 역할을 함께 수행한다.
    - 트랜잭션 개념이 없다.
    - 하둡 맵리듀스 개념이다.
    - 데이터 저장소로 스펙트럼(S3) 저장소를 사용할 경우 업데이트, DELETE 개념이 없이 쓰기 개념만 있다.
    - SQL 엔진은, 커스커마이징 된 Postgresql 엔진을 사용한다.
    - 최하스펙 (cpu2,  15GB memory)노드 1개 유지비용은 하루 8,000 원 이다.  [가격정보](https://aws.amazon.com/ko/redshift/pricing/)
    - 인덱스 개념이 없고, 유일한 인덱스는 파티션 키이다. 그러나 시스템에서는 job 또는 date 단위를 이미 파티션 키로 쓰고 있으므로 임의의 컬럼에 대해 order by 를 사용시 데이터 양에 비례하여 급격히 느려진다.
  - Compute Node	
    - 리더노드로부터 쿼리플랜을 할당받아 S3 에서 데이터를 스캔하여 작업하거나, S3 에 결과를 인서트한다.
    - 스펙트럼(S3) 저장소 사용시 데이터가 파티션 처리되어있을 경우 성능이 올라간다.
    - 스펙트럼(S3) 저장소 사용시 저렴한 노드로 다수로 구성되어 있을 경우 파티션된 파일에 대한 병렬처리가 가능하므로 성능이 노드수에 비례하여 올라간다.

- Application
  - Batch	
    - RDS 로부터 매일의 기준데이터를 텍스트파일로 만들어 S3 에 업로드한다.  RDBMS 에 인서트하지 않고 S3 업로드이므로 상대적으로 빨리 끝난다.
  - 관리형 변수	
    - 지속적인 시간을 두고 데이터 업데이트가 필요한 변수이다.
    - 프로세스가 시작될 때 분석일수 만큼의 정보를 해당 프로세스 작업의 변수로 Redshift 에 업로드 되고, 스냅샷 형식으로 남게된다.
    - 그러므로 관리형 변수를 계속 변경 하더라도 하나의 작업에서 사용된 변수들은 작업시점에 사용된 값으로 조회가 가능하다.
  - 즉시성 변수	
    - 프로세스가 시작될때 입력받는 즉시성 변수로써,  프로세스가 생성하는 쿼리문과 로직에 영향을 미친다.
    - 프로세스 변수테이블에 저장된다.

- Lambda	
  - 엑셀작업	
    - Java 로 작성된 Lambda 모듈을 빌드할 경우 Docker 이미지가 생성되고 Jenkins 를 통해 AWS Lambda 로 배포된다.
    - BPMN 엔진이 각 프로세스 단계마다 Labmda 직접 실행시키며,  Lambda 에서는 Redshift 에 엑셀생성을 위한 쿼리를하고,  엑셀 생성 후 S3 에 업로드한다.

# 아쉬웠던 점

프로젝트 종료 후 아쉬웠던 점이 몇가지 남았습니다. 시간을 충분히 들여 공부했다면 좀 더 가벼운 그림으로 접근할 수 있었을 텐데 하는 생각이 드는 점입니다.

## Glue ETL 을 좀더 활용하지 못하였다.

매일 기준데이터를 불러오는 부분을, Spring Batch 를 통해 구현하였습니다. Spring Batch 는 정적인 Jenkins 서버에서 많은 양의 데이터를 프로세싱 하기 위해 청크단위로 작업을 나누어 실행하는 전통적인 방식입니다. Glue ETL 을 활용하여 Batch 어플리케이션을 직접 만들지 않고 AWS 에 모두 위임하고 싶었지만, 한번도 해보지 않은 사항에 대한 위험성과, Glue ETL 로 하여금 운영 RDS 로 접근할때의 보안이슈 가 있을것이라 예상되어 추진하지는 못했습니다. 기술적으로 주요 credential 을 보호할 수 있을지는 몰라도, 보안을 적용시키기 위한 행적적인 절차의 부담이 크기 때문입니다.

## Aws Step Functions workflow 를 활용하지 못한 점.

Aws Step Functions workflow 은 프로젝트 말미에 유튜브에서 aws services korea 채널을 보다가 Redshift 활용사례에 관한 몇가지 영상을 보고서 알게 된 사실입니다. 이것은 이 프로젝트에서 도입했던 BPMN 엔진을 AWS 상에서 구현할 수 있게 해 줍니다!
아마 프로젝트 초반에 이것을 알았다면 어느것을 선택해야 할지 꽤나 고민했을 것 입니다. 제 성격상 새로운 사례를 만들어 내기 좋아하기 때문에 Aws Step Functions 를 파고들어갈 것이 분명하지만... 
대충 다음의 사유를 들어서 아마 힘들었을 것이라고 위안을 삼아 봅니다.

- 업무 프로세스와의 연속성 측면이나 승인과정(Approval) 처리를 위해 부가적으로 관리해야 하는 AWS 컴포넌트가 계속 늘어났을 것이다.
- 프로세스 속의 복잡한 업무 비즈니스를 Lambda 로 이전해야 할 때의 부담감. (수많은 디펜던시 코드가...)
- 더 유연한 프로세스 설계는 BPMN 엔진이 더 우세함.

** -끝- **