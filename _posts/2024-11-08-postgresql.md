---
layout: post
title:  "3. K8s환경에서 Postgresql 사용하기"
nav_exclude: true
author: kyehuijun
categories: [ OSS ]
image: assets/images/thumbnail/oss-hcp.png
featured: true
rating: 0.0
---

# 1. 소개
- PostgreSQL은 오픈 소스 객체-관계형 데이터베이스 시스템으로, SQL 표준(ANSI SQL)을 준수하고 최신 SQL 기능을 지원한다.
   > 객체-관계형 데이터베이스 시스템(Object-Relational Database Management System, ORDBMS): 관계형 데이터베이스 관리 시스템(RDBMS)과 객체 지향 데이터베이스 관리 시스템(ODBMS)의 특성을 결합한 데이터베이스 시스템
- 관계형 데이터뿐만 아니라 JSON 타입의 데이터를 저장하고 쿼리할 수 있는 기능을 제공한다.
- OSS 설치 시 Metadata를 관리하기 위한 용도로 `Postgresql`을 사용하기 위해 설치한다.

## 1.1. 구성요소

![]({{ site.baseurl }}/assets/images/2024-11-08-postgresql/postgresql.drawio.png)

1. Shared Memory
- Share Memory에서 가장 중요한 요소는 Shared Buffer와 WAL Buffer 이다.
- pgAdmin을 사용하면 관리자는 버퍼의 활용도와 성능에 대한 insight을 얻을 수 있어 효율적인 관리에 도움이 됩 수 있다.
   -  Shared Buffer
       - Shared Buffer 목표는 모든 데이터베이스가 그렇듯 disk I/O를 최소화 하는 것이다. 
       - 이를 위해 하기 항목을 만족해야 한다.
           1. 매우 큰(수십 또는 수백 GB) 버퍼에 빠르게 액세스해야 한다.
           2. 많은 사용자가 동시에 접속할 때 경합을 최소화한다.
           3. 자주 사용되는 블록은 가능한 한 오랫동안 버퍼에 있어야 한다.

   - WAL Buffer
       - WAL Buffer는 데이터의 변경사항을 잠시 저장하는 버퍼이다.
       - WAL Buffer에 기록된 데이터는 정해진 시점에 WAL 파일로 기록된다.
       - 백업 및 복구 관점에서 WAL 버퍼와 WAL 파일은 매우 중요하다. (오라클의 Redo Log Buffer 및 Redo Log File과 유사)
       - pgAdmin을 사용하는 경우 이를 사용하여 기록되는 데이터의 빈도와 양을 포함하여 WAL 버퍼의 활동을 모니터링할 수 있다.

2. Postmaster (Daemon) Process
   - PostgreSQL 기동할 때 가장 먼저 시작되는 프로세스이다.
   - 초기 복구 작업, 메모리 초기화, Background 프로세스 기동작업 수행한다.
   - 데몬 프로세스로 Client 프로세스의 접속 요청을 받아 Backend 프로세스를 생성한다.
   - 또한, Postmaster 프로세스는 제일 앞단에서 Client와 맞닫는 프로세스이다.

   ![]({{ site.baseurl }}/assets/images/2024-11-08-postgresql/postgresql_backend_process.drawio.png)

   - Background Processes는 로그, 통계 등 백그라운드에서 배치성으로 돌아가는 프로세스

   | Process | Function |
   |--|--|
   |logger|에러 메시지를 로그 파일에 기록한다.|
   |checkpointer|체크 포인트 발생시 dirty 버퍼를 데이터파일에 기록한다.|
   |Background writer|로그 및 백업 정보를 최신 상태로 유지.|
   |wal writer|WAL 버퍼를 WAL 파일에 기록한다.|
   |autovacuum launcher|Vacuum 이 필요한 시점에 Postmaster 프로세스에 autovacuum worker 프로세스 기동 요청한다.|
   |archiver|Archive mode 사용시 WAL 파일을 지정된 디렉토리로 복사한다.|
   |stats collector|세션 수행 정보, 테이블 사용 통계 정보 같은 DBMS 통계 정보 수집한다.|

   - Backend Process
       - Client가 요청한 SQL 및 Command를 처리하는 프로세스로 Client와의 연결이 끊어지면 종료된다.
       - postmaster 프로세스에 의해 시작되며 Client 와는 1:1 관계이다.
       - Max Connection 수치만큼 Client가 동시에 연결 될 수 있다, (Deafult 100)

### ref.
- https://www.postgresql.org/docs/current/tutorial-arch.html
- https://snowturtle93.github.io/posts/Postgresql-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-%EB%B0%8F-%ED%8A%B9%EC%A7%95/
- https://medium.com/agedb/postgresql-architecture-59d6242d91d8
- https://blog.ex-em.com/1645

# 3. Postgresql 설치
- [binami helmchart](https://github.com/bitnami/charts) 를 기준으로 하며, 내부 환경에 Image Import는 [링크](https://hcp-oss.github.io/private-container-registry/)를 참조하고 Chart Import는 [링크](https://hcp-oss.github.io/private-helm-chart/)를 참조한다.
- values.yaml 파일은 하기와 같이 구성한다.

```yaml
global:
  defaultStorageClass: {{PostgresqlStorageClassName}} # Postgresql 에서 사용할 Storage Class 이름

image:
  registry: {{ImageRegistry}} # Private Container Registry, public 환경은 docker.io를 입력한다.
  repository: bitnami/postgresql # Private Container Registry의 Image Repository, public 환경은 bitnami/postgresql를 입력한다.
  tag: {{ImageVsersion}} # 사용할 Image의 Tag

auth:
  enablePostgresUser: true  # admin 계정 사용여부, 다른 사용자 계정 및 Database 생성을 위해 활성화한다.
  postgresPassword: {{PostgresqlAdminPassword}} # admin 계정의 Password

  secretKeys:
    adminPasswordKey: postgres-password  # admin 계정의 Password가 저장될 Secret

```
> 이외에 설정 가능한 항목은 Bitnami chart의 values.yaml 파일을 참조한다.

- Helm릉 통해 설치한다.

# 4. Postgresql 접속 및 테스트

# 5. 사용자 및 DB 추가