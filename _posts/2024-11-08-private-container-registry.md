---
layout: post
title:  "1. Private Container Registry 사용하기"
nav_exclude: true
author: kyehuijun
categories: [ OSS ]
image: assets/images/thumbnail/oss-hcp.png
featured: true
rating: 0.0
---

# 1. 소개
- 일반적인 public 환경에서는 container image를 docker hub 혹은 gcr 등에서 가져올 수 있다.
- 그러나 사내망은 private 환경인 경우가 많으며, 이 경우 private container registry를 구성해서 사용한다.
- public cloud 내에 private network 등을 구성해서 사용한다면 각 CSP에서 제공하는 Container Registry 서비스를 사용할 수 있다. ( Azure의 ACR, AWS의 ECR 등 )
- public cloud를 사용하지 못한다면, 자체적으로 사내에 구축해서 사용할 수 있다. (harbor 등)
- 본 포스트에서는 docker hub에서 이미지를 pull, AWS에 구축한 ECR(Elastic Container Registry)에 push하는 과정을 정리한다.

* **ECR이 아니어도 하기 내용의 경로만 달리 하면 된다. (수행 과정에 차이가 있지 않다.)**


# 2. Container Image Pull/Push
- 본 포스트에서는 Image Pull push를 위해 `podman`을 사용한다.
> Podman : deamon-less 오픈소스 Container Engine으로, OCI 표준을 따르기 떄문에 Docker 와 호환 가능하다.

1. podman설치 
- https://podman.io/docs/installation

1. `images.txt`파일 생성 후 image 목록 작성

```bash
# images.txt 예시
docker.io/library/nginx:latest
docker.io/library/alpine:latest
docker.io/library/redis:latest
```

2. `imagePullPush.sh`파일 생성 후 하기 내용 작성

```bash
#!/bin/bash

# 설정: 변경할 변수들
IMAGE_LIST_FILE="images.txt" # Image 목록이 저장된 파일 이름
PRIVATE_REGISTRY= {{PriavteRegistryUrl}}    # private registry 주소
REGISTRY_USER= {{PriavteRegistryUser}}   # registry 사용자명
REGISTRY_PASS= {{PriavteRegistryPassword}}   # registry 비밀번호 

# Private registry 로그인
echo "Private registry에 로그인 중..."
podman login "$PRIVATE_REGISTRY" -u "$REGISTRY_USER" -p "$REGISTRY_PASS"
if [[ $? -ne 0 ]]; then
    echo "Error: Registry 로그인에 실패했습니다."
    exit 1
fi

# 이미지 목록 파일에서 IMAGES 배열로 읽어오기
if [[ ! -f $IMAGE_LIST_FILE ]]; then
    echo "Error: $IMAGE_LIST_FILE 파일을 찾을 수 없습니다."
    exit 1
fi

mapfile -t IMAGES < "$IMAGE_LIST_FILE"

# 이미지 목록 pull, private registry로 push
for IMAGE in "${IMAGES[@]}"; do
    # 이미지 pull
    podman pull "$IMAGE"

    # 이미지 이름과 태그 분리
    IMAGE_NAME=$(echo "$IMAGE" | cut -d':' -f1)
    IMAGE_TAG=$(echo "$IMAGE" | cut -d':' -f2)

    # 새로 tag 할 이미지 이름 지정
    NEW_IMAGE="$PRIVATE_REGISTRY/$IMAGE_NAME:$IMAGE_TAG"

    # 이미지 tag 변경
    podman tag "$IMAGE" "$NEW_IMAGE"

    # private registry로 push
    podman push "$NEW_IMAGE"
    echo "Successfully pushed $NEW_IMAGE to $PRIVATE_REGISTRY"

    # 사용이 끝난 이미지 삭제
    podman rmi "$IMAGE"
    podman rmi "$NEW_IMAGE"
    echo "Deleted local images: $IMAGE and $NEW_IMAGE"
done

echo "모든 이미지를 private registry로 업로드 완료!"
```
> 인증서 오류가 발생할 경우 login, pull, push 명령어에 `--tls-verify=false`을 추가한다.

3. `ImagePullPush.sh`파일 실행

```
/bin/bash imagePullPush.sh
```

4. 결과 확인
![]({{ site.baseurl }}/assets/images/2024-11-08-private-container-registry/container_image_upload_result.png)