+++
title = "docker? containerd? runc? 이게 다 뭐지? podman은 또 뭐고?"
description = "container 관련 software들을 나열하고 알아보자"
date = "2023-11-14"
taxonomies.tags = ["container", "devops", "foobar"]
+++

> 이 글은 제 개인 Discord server에서 [김지현 (simnalamburt)](https://hyeon.me) 님이 설명해주신 내용을 정리한 것입니다.

Linux container의 세계를 여행하다보면, 여러가지 software들이 등장합니다.
docker, containerd, runc, podman, CRI-O, nerdctl, ctr, crictl, ...
이 많은 software들은 다 어떤 차이와 쓰임새가 있고, 왜 등장하게 된 걸까요?
container와 관련된 여러 software들을 나열하고 살펴봅시다.

## docker

태초에는 docker가 있었습니다.
docker는 software로써 성공하기 위해 편의성 기능도 많이 넣고, 처음에는 가상화였다가 lxc로 바꿨다가 cgroup으로 바꾸는 등의 일도 전부 docker라는 한 이름 아래에서 일어났습니다.
LXC의 등장 이전까지의 역사와 docker에 대한 자세한 설명은 이 글에서 생략합니다.

## OCI (Open Container Initiative)

docker라는 가상화 software가 흥하니, CoreOS의 rkt 같은 경쟁자들도 생겨납니다.

이 때 docker는 표준화 과정을 딱히 거치지 않았습니다. container image 형식도 docker 자신만의 비표준 형식을 사용하고 있었습니다.
그에 반해 rkt는 appc (application container)라는 container image 형식을 사용했습니다.

그러나 rkt는 docker와 호환이 되지 않고, docker가 더욱 득세하면서 rkt는 망하게 됩니다.

rkt가 망하기 전, rkt 개발자들은 docker 개발자들에게 container라는 기술은 앞으로 발전할 것이므로, 우리 공개된 container 표준을 하나 만들자고 열심히 설득합니다.
비록 rkt는 망하게 되었지만, 이 공개된 container 표준은 남아서 성공하게 되고, 우리는 이를 OCI (Open Container Initiative) 표준이라고 부릅니다.

## runc

docker가 OCI를 구현할 당시, docker의 codebase가 너무 크고 복잡해져서 OCI를 구현하는 부분을 독립 project로 찢어내자는 의견이 득세하고 실행되게 됩니다.
그래서 OCI 표준을 구현하고, 실제로 container를 실행하는 부분이 runc라는 이름으로 찢어지게 됩니다.

우리가 container의 구조에 대해 deep dive할 때 배우는 cgroup과 소통하는 등의 일은 전부 실제로는 runc가 하게 됩니다.

docker와, 이후 나올 CRI-O, podman 등 대부분의 container runtime이 실제로 container를 실행하는 데에는 runc를 사용합니다.
container를 실행할 때 마다 runc를 하나씩 새로 켜는 방식입니다.

crun이라는 minor한 OCI 구현체도 있지만 거의 사용되지 않습니다.

## CRI (Container Runtime Interface)

Kubernetes는 원래 container를 실행하는 부분으로 docker를 쓰도록 hard-coding 되어있다가, rkt가 등장하면서 docker과 rkt를 고를 수 있게 했다가, rkt가 망하고 다른 구현체가 등장하면서 이번에는 hard-code하지 말고 docker 등의 container runtime을 interface로 추상화하자고 생각하게 됩니다.
이것이 바로 CRI (Container Runtime Interface) 입니다.

## CRI-O

docker는 docker-compose, docker-swarm 등이 포함되고 편의성을 위해 이것저것 집어넣은 거대한 software입니다.
OCI와 runc가 등장한 이후 runc만 감싼 lightweight한 새 container manager를 만들 수 있겠다고 생각되어, 이런 software가 여럿 등장했는데 그 중 하나가 CRI-O 입니다.

Kubernetes의 경우 container의 관리는 실제로는 Kubernetes가 하고, CRI 구현체는 container를 적당히 켜고 끄는 일만 하면 됩니다.
여기에 docker는 너무 거대한 software입니다.
그래서 runc를 이용해 CRI 표준만 구현한 아주 가벼운 software가 CRI-O 입니다.

## containerd

어차피 docker 개발자나 Kubernetes 개발자나 다 겹치는 상황 아래에서 Kubernetes에서 CRI 논의를 할 때, Kubernetes의 hard-coded docker를 제거할 때 docker도 CRI로 가져다 쓸 수 있으면 좋겠다고 생각하게 됩니다.

docker 개발자들은 docker 위에 CRI를 구현하느니, OCI 표준을 구현할 때 runc를 찢어낸 것 처럼, docker 안에 CRI 표준에 해당하는 작은 부분을 찢어내자고 생각하게 됩니다.
이것이 실행된 것이 containerd입니다.

원래는 docker가 runc를 직접 실행하는 구조였는데, docker daemon과 containerd daemon이 같이 실행되고, docker가 containerd에 명령을 내리면 containerd가 runc를 실행하는 구조로 바뀝니다.

## ctr

ctr은 containerd용 CLI입니다.

## crictl

crictl은 CRI interface를 구현하는 containerd, CRI-O 등을 조작하는 CLI입니다.

## nerdctl

ctr과 crictl은 사람이 쓰기에 많이 불편하고, 실제로 사람이 쓰는 것을 가정하고 만들어진 것도 아닙니다.
그래서 등장한 것이 nerdctl입니다.

nerdctl은 containerd 용 CLI인데, docker와 UX를 비슷하게 만들어서 사용하기 편합니다.

## podman

비록 rkt는 망했지만, rkt도 여러 장점이 있었습니다.
rkt는 docker와는 다르게 daemon이 계속 떠있을 필요도 없고, sudo 권한 없이 rootless로 container를 실행하는 것도 가능했습니다.

rkt를 그리워하던 사람들이, 시대가 좋아져 OCI, CRI 등의 표준도 잘 잡혔으니, 다시 docker와 완전히 호환되고 daemon 없이 실행가능하면서 rootless container도 지원하는 runtime을 만들어보자고 한 것이 podman입니다.

podman도 위에서 설명한 것 처럼 내부적으로는 결국 runc를 사용합니다.

또한 podman은 CRI도 구현합니다.

그래서 podman은 docker와 CLI UX가 비슷하다는 점에서 docker와 같은 수준이지만, CRI 구현체라는 점에서 containerd, CRI-O와 같은 수준이기도 합니다.

docker는 container image를 관리하는 부분은 [containers/image](https://github.com/containers/image)로, filesystem mount 관련 부분은 [containers/storage](https://github.com/containers/storage)로 찢어졌는데, podman은 이 둘을 그대로 사용합니다.
그래서 docker와 podman은 사실 많은 양의 codebase를 공유합니다.

## libpod

podman 내부에서 runc를 실행하는 부분을 libpod이라는 이름으로 찢는 시도가 현재 진행 중입니다.
별도로 program을 설치하는 것이 아니라 Go library 함수를 호출하면 container가 올라가고 내려가는 등을 할 수 있게 하는 것이 목표입니다.

## [youki](https://github.com/containers/youki)

youki는 Rust로 짜여진 OCI runtime 표준 구현체로, runc와 crun과 같은 수준에 있는 software입니다.
