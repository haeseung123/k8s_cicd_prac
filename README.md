# Kustomize를 활용한 Manifest 관리

이 레포지토리는 Kustomize를 사용하여 쿠버네티스의 리소스(Manifest)들을 효율적으로 관리하는 방법을 공부하고 적용한 경험을 기록했습니다.

새로운 버전으로 이미지를 교체하여 배포하는 과정에서 발생한 문제들을 해결한 방법도 함께 정리하였습니다.

## 프로젝트 구조

```
├── base
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays
    ├── development
    │   ├── kustomization.yaml
    │   └── deployment-patches.yaml
```

##

### overlays/kustomization.yaml의 images 필드와 patches 필드 중복 이슈

`images` 필드와 `patches`를 동시에 사용하여 Deployment 리소스 내의 이미지를 명시적으로 수정했습니다. 하지만 Kustomize에서는 이 두 가지 방법이 충돌할 수 있다는 점을 알게 되었습니다.

`images` 필드를 사용하여 이미지를 변경하는 것과 `patches`를 사용하여 Deployment 리소스 내에서 이미지를 명시적으로 수정하는 것은 사실상 중복된 작업입니다.

구체적으로, `patches`와 `images`는 모두 Kustomize에서 리소스를 수정하는 데 사용되지만 사용하는 목적과 방식에 차이가 있습니다.

- patches: 이미 정의된 Kubernetes 리소스의 특정 부분을 수정하는 데 사용되는데 이미 존재하는 리소스의 필드나 값을 덮어씌우는 방식

- Images: 이미지를 변경하는 데 특화된 필드로 이미지와 관련된 값만을 변경하고자 할 때 사용하며 기존 Deployment나 Pod 리소스의 image 필드를 변경

Kustomize에서 `images` 필드와 `patches` 필드를 동시에 사용하여 동일한 리소스의 이미지를 변경하려고 할 때 충돌로 인해 이미지 변경이 예상대로 적용되지 않는 문제가 발생할 수 있습니다.

특히 이미지 버전에 차이가 없을 경우 문제 발생 가능성은 낮지만 버전이 다를 경우 충돌을 방지하려면 하나의 방법을 선택하는 것이 좋습니다.

_관련된 문제는 [이슈 #4492](https://github.com/kubernetes-sigs/kustomize/issues/4492)를 참고_

### Kustomize의 우선순위 이해

`images` 필드와 `patches` 각각 다른 버전으로 적용하여 우선순위 적용 테스트를 진행해보았습니다.

#### 1.

- overlays/development/kustomization.yaml

```
images:
  - name: haeseung/kanban-server
    newTag: v1.0.22
```

- overlays/development/deployment-patches.yaml

```
image: haeseung/kanban-server:v1.0.23
```

적용 시 **Container image "haeseung/kanban-server:v1.0.22"** 이벤트 로그 확인

#### 2.

- overlays/development/kustomization.yaml

```
images:
  - name: haeseung/kanban-server
    newTag: v1.0.23
```

- overlays/development/deployment-patches.yaml

```
image: haeseung/kanban-server:v1.0.22
```

적용 시 **Container image "haeseung/kanban-server:v1.0.23"** 이벤트 로그 확인

<br>

즉, patches 필드가 먼저 적용되고 이후 images 필드가 덮어 씌워지는 방식으로 확인되었으며 따라서 Kustomize에서는 patches 필드가 우선 적용된다고 볼 수 있습니다.

images와 patches 필드가 충돌하는 경우 Kustomize는 명시적으로 설정된 값을 우선시합니다. 이로 인해 이미지가 변경되지 않거나 의도한 대로 적용되지 않는 문제가 발생할 수 있습니다.

### 결론

여러 설정 방법을 동시에 사용할 때 발생할 수 있는 충돌을 피하려면 각 필드의 역할과 우선순위를 명확히 이해하는 것이 중요합니다.

이 프로젝트의 경우 `images` 필드를 주석처리하고 `patches` 하나만 사용하는 것으로 문제를 해결하였습니다.
