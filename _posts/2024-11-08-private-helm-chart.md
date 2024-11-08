---
layout: post
title:  "2. private Helm Chart Repository 사용하기"
nav_exclude: true
author: kyehuijun
categories: [ OSS ]
image: assets/images/thumbnail/oss-hcp.png
featured: true
rating: 0.0
---

# 1. 소개
- 일반적인 public 환경에서는 `https://kubernetes.github.io/ingress-nginx` 등 제공 업체의 공식 Helm Repository를 구성해서 사용한다.
- 그러나 사내망은 private 환경인 경우가 많으며, 이 경우 Local에 Helm chart를 저장해 사용하거나, private helm chart repository 구성해서 사용한다.
- 본 포스트에서는 private helm chart repository 에 chart를 upload하고, 배포하는 과정을 설명한다.

- **Harbor의 경우 chart meusium을 통해 chart를 관리할 수 있으며, 추후 Harbor 구성 후 포스트 할 예정이다.**

# 2. Helm Chart 준비
- private helm chart repository에 업로드할 helm chart를 다운 받는다. 
- 본 포스트에서는 bitnami의 postgresql chart를 업로드 하는 것을 예시로 설명한다. (https://github.com/bitnami/charts)
- `Chart.yaml` 파일의 내용을 확인하여 dependency를 확인힌다.

```yaml
# Copyright Broadcom, Inc. All Rights Reserved.
# SPDX-License-Identifier: APACHE-2.0

annotations:
  category: Database
  licenses: Apache-2.0
  images: |
    - name: os-shell
      image: docker.io/bitnami/os-shell:12-debian-12-r32
    - name: postgres-exporter
      image: docker.io/bitnami/postgres-exporter:0.15.0-debian-12-r45
    - name: postgresql
      image: docker.io/bitnami/postgresql:17.0.0-debian-12-r11
apiVersion: v2
appVersion: 17.0.0
dependencies:
- name: common
  repository: oci://repository-1.docker.io/bitnamicharts
  tags:
  - bitnami-common
  version: 2.x.x
description: PostgreSQL (Postgres) is an open source object-relational database known for reliability and data integrity. ACID-compliant, it supports foreign keys, joins, views, triggers and stored procedures.
home: https://bitnami.com
icon: https://bitnami.com/assets/stacks/postgresql/img/postgresql-stack-220x234.png
keywords:
- postgresql
- postgres
- database
- sql
- replication
- cluster
maintainers:
- name: Broadcom, Inc. All Rights Reserved.
  url: https://github.com/bitnami/charts
name: postgresql
sources:
- https://github.com/bitnami/charts/tree/main/bitnami/postgresql
version: 16.1.2

```

- 위 `Chart.yaml` 에서 유심히 봐야할 부분은 `dependencies` 이다.
   >  `dependencies` 는 helm chart 설치를 위해 다른 helm chart에 대한 의존성을 명시하는 것으로, Private repository에 업로드 하기 위해 helm을 package 하거나, helm chart를 install 하기위해 필요한 또 다른 chart 이다.
- 위 `Chart.yaml`의 `dependencies`에 명시된 common chart는 bitnami의 다른 chart(e.g. keycloak)와 함께 설치될 떄 `values.yaml`의 변수를 변경하지 않고 사용하기 위해 제공되는 chart이다.
   > common 차트 내부에서 변수를 통일해주는 역할을 한다.
- common chart의 `repository`를 확인하면 bitnami의 repository를 참고하고 있는데, 이는 private 환경에서 접근하지 못할 수 있다.
- 그래서 private 환경이라면 가장 먼저 의존성이 없는 chart인 common chart를 private repository에 import 한다.
- 그 후 common chart에 의존성이 있는 postgresql chart를 import 한다.

# 3. Helm chart 설치에 사용할 변수 설정
- 가장 먼저 앞으로 사용할 변수를 설정한다.
- Public 혹은 Private 환경에 맞게 환경변수를 설정한다.

```bash
# Helm Chart Pull/Push 관련
REPOSITORY_NAME={{PriavteRegistryName}} # repository 이름 
REPOSITORY_URL={{PriavteRegistryUrl}}    # repository 주소
REPOSITORY_USER={{PriavteRegistryUser}}   # repository 사용자명
REPOSITORY_PASS={{PriavteRegistryPassword}}   # repository 비밀번호 

# Helm Chart 설치 관련
CHART_NAME={{CharReleaseName}}    # chart releas 이름
RELEASE_NAME={{CharReleaseName}}    # chart releas 이름
CUSTOM_VALUE_FILE_NAME=custom-values.yaml    # chart에 사용할 values 파일 이름
K8S_NAMESPACE={{NAMESPACE}} # K8s Namespace
```

# 4. Helm chart Impoer
- public 환경이라면 4번 과정은 skip해도 된다.
- **(Option 1)** Repository가 OCI 규격을 사용하는 저장소일 경우

```bash
REMOVED_OCI_REPOSITORY=$(echo "$REPOSITORY_URL" | sed 's|^oci://||')

# Private repository에 로그인
helm registry login $REMOVED_OCI_REPOSITORY --username "$REPOSITORY_USER" --password "$REPOSITORY_PASS"
```

- **(Option 2)** Repository가 OCI 규격을 사용하는 저장소가 아닐 경우

```bash
# Private repository 추가 (username과 password는 없으면 제거해도 된다.)
helm repo add $REPOSITORY_NAME $REPOSITORY_URL --username "$REPOSITORY_USER" --password "$REPOSITORY_PASS"

# helm repository update
helm repo update
```

## 4.1. Chart가 의존성이 없는 경우

- helm chart로 이동해서 하기 명령어 실행

```bash
# Helm chart Packaged
helm package .

# packaged 된 파일 이름 확인
PACKAGED_FILE_NAME=$(ls -l | grep tgz | awk '{print $9}')

# 출력
echo $PACKAGED_FILE_NAME

# helm push
helm push $PACKAGED_FILE_NAME $REPOSITORY_URL
```

- 결과 확인

![]({{ site.baseurl }}/assets/images/2024-11-08-private-helm-chart/chart_push_result.png)

## 4.2. Chart가 의존성이 있는 경우
- 우선 `Chart.yaml`에서 `dependeny`의 URL을 private repository 주소로 변경해야 한다.
- 예시는 다름과 같다.

```yaml
...
dependencies:
- name: common
  repository: oci://533616270150.dkr.ecr.ap-northeast-2.amazonaws.com/kye/chart
  tags:
  - bitnami-common
  version: 2.x.x
...
```
- 하기 명령어를 통해 변경한다.

```bash
#
sed -i 's\repository: .*\repository: '$REPOSITORY_URL'\g' Chart.yaml
```
> Mac의 경우 `sed -i '' "s|repository: .*|repository: ${REPOSITORY_URL}|" Chart.yaml`

- 그후 Helm Package를 및 Upload를 진행한다.

```bash
# chart에 의존성 update
helm dependency update

# Helm chart Packaged
helm package .

# packaged 된 파일 이름 확인
PACKAGED_FILE_NAME=$(ls -l | grep tgz | awk '{print $9}')

# 출력
echo $PACKAGED_FILE_NAME

# helm push
helm push $PACKAGED_FILE_NAME $REPOSITORY_URL
```

# 5. Helm 설치
- values 파일 작성

```bash
# custom values file 작성
vim $CUSTOM_VALUE_FILE_NAME

# 각 chart에 맞는 value 설정 후 저장
```
- **(Option 1)** Repository가 OCI 규격을 사용하는 저장소일 경우

```bash
# Helm install
helm upgrade $RELEASE_NAME $REPOSITORY_URL/$CHART_NAME \
    -i \
    --namespace $K8S_NAMESPACE \
    --create-namespace \
    -f $CUSTOM_VALUE_FILE_NAME

```

- **(Option 2)** Repository가 OCI 규격을 사용하는 저장소가 아닐 경우

```bash
# Helm install
helm upgrade $RELEASE_NAME $REPOSITORY_NAME/$CHART_NAME \
    -i \
    --namespace $K8S_NAMESPACE \
    --create-namespace \
    -f $CUSTOM_VALUE_FILE_NAME
```

- 설치가 되었는지 확인한다.

```bash
# helm chart install 되었는지 확인
helm ls -n $K8S_NAMESPACE

# 실제로 자원이 할당 되었는지 확인한다.
kubectl get sts -n $K8S_NAMESPACE
```