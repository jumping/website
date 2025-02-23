---



title: 로깅 아키텍처
content_type: concept
weight: 60
---

<!-- overview -->

애플리케이션 로그는 애플리케이션 내부에서 발생하는 상황을 이해하는 데 도움이 된다. 로그는 문제를 디버깅하고 클러스터 활동을 모니터링하는 데 특히 유용하다. 대부분의 최신 애플리케이션에는 일종의 로깅 메커니즘이 있다. 마찬가지로, 컨테이너 엔진들도 로깅을 지원하도록 설계되었다. 컨테이너화된 애플리케이션에 가장 쉽고 가장 널리 사용되는 로깅 방법은 표준 출력과 표준 에러 스트림에 작성하는 것이다.

그러나, 일반적으로 컨테이너 엔진이나 런타임에서 제공하는 기본 기능은 완전한 로깅 솔루션으로 충분하지 않다.
예를 들어, 컨테이너가 크래시되거나, 파드가 축출되거나, 노드가 종료된 경우에도 애플리케이션의 로그에 접근하고 싶을 것이다.
클러스터에서 로그는 노드, 파드 또는 컨테이너와는 독립적으로 별도의 스토리지와 라이프사이클을 가져야 한다. 이 개념을 _클러스터-레벨-로깅_ 이라고 한다.  

<!-- body -->

클러스터-레벨 로깅은 로그를 저장하고, 분석하고, 쿼리하기 위해 별도의 백엔드가 필요하다. 쿠버네티스는
로그 데이터를 위한 네이티브 스토리지 솔루션을 제공하지 않지만,
쿠버네티스에 통합될 수 있는 기존의 로깅 솔루션이 많이 있다.

## 쿠버네티스의 기본 로깅

이 예시는 텍스트를 초당 한 번씩 표준 출력에 쓰는
컨테이너에 대한 `Pod` 명세를 사용한다.

{{< codenew file="debug/counter-pod.yaml" >}}

이 파드를 실행하려면, 다음의 명령을 사용한다.

```shell
kubectl apply -f https://k8s.io/examples/debug/counter-pod.yaml
```

출력은 다음과 같다.

```console
pod/counter created
```

로그를 가져오려면, 다음과 같이 `kubectl logs` 명령을 사용한다.

```shell
kubectl logs counter
```

출력은 다음과 같다.

```console
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```

`kubectl logs --previous` 를 사용해서 컨테이너의 이전 인스턴스에 대한 로그를 검색할 수 있다. 파드에 여러 컨테이너가 있는 경우, 명령에 컨테이너 이름을 추가하여 접근하려는 컨테이너 로그를 지정해야 한다. 자세한 내용은 [`kubectl logs` 문서](/docs/reference/generated/kubectl/kubectl-commands#logs)를 참조한다.

## 노드 레벨에서의 로깅

![노드 레벨 로깅](/images/docs/user-guide/logging/logging-node-level.png)

컨테이너화된 애플리케이션의 `stdout(표준 출력)` 및 `stderr(표준 에러)` 스트림에 의해 생성된 모든 출력은 컨테이너 엔진이 처리 및 리디렉션 한다.
예를 들어, 도커 컨테이너 엔진은 이 두 스트림을 [로깅 드라이버](https://docs.docker.com/engine/admin/logging/overview)로 리디렉션 한다. 이 드라이버는 쿠버네티스에서 JSON 형식의 파일에 작성하도록 구성된다.

{{< note >}}
도커 JSON 로깅 드라이버는 각 라인을 별도의 메시지로 취급한다. 도커 로깅 드라이버를 사용하는 경우, 멀티-라인 메시지를 직접 지원하지 않는다. 로깅 에이전트 레벨 이상에서 멀티-라인 메시지를 처리해야 한다.
{{< /note >}}

기본적으로, 컨테이너가 다시 시작되면, kubelet은 종료된 컨테이너 하나를 로그와 함께 유지한다. 파드가 노드에서 축출되면, 해당하는 모든 컨테이너도 로그와 함께 축출된다.

노드-레벨 로깅에서 중요한 고려 사항은 로그 로테이션을 구현하여,
로그가 노드에서 사용 가능한 모든 스토리지를 사용하지 않도록 하는 것이다. 쿠버네티스는
로그 로테이션에 대한 의무는 없지만, 디플로이먼트 도구로
이를 해결하기 위한 솔루션을 설정해야 한다.
예를 들어, `kube-up.sh` 스크립트에 의해 배포된 쿠버네티스 클러스터에는,
매시간 실행되도록 구성된 [`logrotate`](https://linux.die.net/man/8/logrotate)
도구가 있다. 애플리케이션의 로그를 자동으로
로테이션하도록 컨테이너 런타임을 설정할 수도 있다.

예를 들어, `kube-up.sh` 가 GCP의 COS 이미지 로깅을 설정하는 방법은
[`configure-helper` 스크립트](https://github.com/kubernetes/kubernetes/blob/{{< param "githubbranch" >}}/cluster/gce/gci/configure-helper.sh)를 통해
자세히 알 수 있다.

**CRI 컨테이너 런타임** 을 사용할 때, kubelet은 로그를 로테이션하고 로깅 디렉터리 구조를 관리한다.
kubelet은 이 정보를 CRI 컨테이너 런타임에 전송하고 런타임은 컨테이너 로그를 지정된 위치에 기록한다.
[kubelet config file](/docs/tasks/administer-cluster/kubelet-config-file/)에 있는
두 개의 kubelet 파라미터 [`containerLogMaxSize` 및 `containerLogMaxFiles`](/docs/reference/config-api/kubelet-config.v1beta1/#kubelet-config-k8s-io-v1beta1-KubeletConfiguration)를
사용하여 각 로그 파일의 최대 크기와 각 컨테이너에 허용되는 최대 파일 수를 각각 구성할 수 있다.

기본 로깅 예제에서와 같이 [`kubectl logs`](/docs/reference/generated/kubectl/kubectl-commands#logs)를
실행하면, 노드의 kubelet이 요청을 처리하고
로그 파일에서 직접 읽는다. kubelet은 로그 파일의 내용을 반환한다.

{{< note >}}
만약, 일부 외부 시스템이 로테이션을 수행했거나 CRI 컨테이너 런타임이 사용된 경우,
`kubectl logs` 를 통해 최신 로그 파일의 내용만
사용할 수 있다. 예를 들어, 10MB 파일이 있으면, `logrotate` 가
로테이션을 수행하고 두 개의 파일이 생긴다. (크기가 10MB인 파일 하나와 비어있는 파일)
`kubectl logs` 는 이 예시에서는 빈 응답에 해당하는 최신 로그 파일을 반환한다.
{{< /note >}}

### 시스템 컴포넌트 로그

시스템 컴포넌트에는 컨테이너에서 실행되는 것과 컨테이너에서 실행되지 않는 두 가지 유형이 있다.
예를 들면 다음과 같다.

* 쿠버네티스 스케줄러와 kube-proxy는 컨테이너에서 실행된다.
* Kubelet과 컨테이너 런타임은 컨테이너에서 실행되지 않는다.

systemd를 사용하는 시스템에서는, kubelet과 컨테이너 런타임은 journald에 작성한다.
systemd를 사용하지 않으면, kubelet과 컨테이너 런타임은 `/var/log` 디렉터리의
`.log` 파일에 작성한다. 컨테이너 내부의 시스템 컴포넌트는 기본 로깅 메커니즘을 무시하고,
항상 `/var/log` 디렉터리에 기록한다.
시스템 컴포넌트는 [klog](https://github.com/kubernetes/klog)
로깅 라이브러리를 사용한다. [로깅에 대한 개발 문서](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-instrumentation/logging.md)에서
해당 컴포넌트의 로깅 심각도(severity)에 대한 규칙을 찾을 수 있다.

컨테이너 로그와 마찬가지로, `/var/log` 디렉터리의 시스템 컴포넌트 로그를
로테이트해야 한다. `kube-up.sh` 스크립트로 구축한 쿠버네티스 클러스터에서
로그는 매일 또는 크기가 100MB를 초과하면
`logrotate` 도구에 의해 로테이트가 되도록 구성된다.

## 클러스터 레벨 로깅 아키텍처

쿠버네티스는 클러스터-레벨 로깅을 위한 네이티브 솔루션을 제공하지 않지만, 고려해야 할 몇 가지 일반적인 접근 방법을 고려할 수 있다. 여기 몇 가지 옵션이 있다.

* 모든 노드에서 실행되는 노드-레벨 로깅 에이전트를 사용한다.
* 애플리케이션 파드에 로깅을 위한 전용 사이드카 컨테이너를 포함한다.
* 애플리케이션 내에서 로그를 백엔드로 직접 푸시한다.

### 노드 로깅 에이전트 사용

![노드 레벨 로깅 에이전트 사용](/images/docs/user-guide/logging/logging-with-node-agent.png)

각 노드에 _노드-레벨 로깅 에이전트_ 를 포함시켜 클러스터-레벨 로깅을 구현할 수 있다. 로깅 에이전트는 로그를 노출하거나 로그를 백엔드로 푸시하는 전용 도구이다. 일반적으로, 로깅 에이전트는 해당 노드의 모든 애플리케이션 컨테이너에서 로그 파일이 있는 디렉터리에 접근할 수 있는 컨테이너이다.

로깅 에이전트는 모든 노드에서 실행해야 하므로, 에이전트는
`DaemonSet` 으로 동작시키는 것을 추천한다.

노드-레벨 로깅은 노드별 하나의 에이전트만 생성하며, 노드에서 실행되는 애플리케이션에 대한 변경은 필요로 하지 않는다.

컨테이너는 stdout과 stderr를 동의되지 않은 포맷으로 작성한다. 노드-레벨 에이전트는 이러한 로그를 수집하고 취합을 위해 전달한다.

### 로깅 에이전트와 함께 사이드카 컨테이너 사용 {#sidecar-container-with-logging-agent}

다음 중 한 가지 방법으로 사이드카 컨테이너를 사용할 수 있다.

* 사이드카 컨테이너는 애플리케이션 로그를 자체 `stdout` 으로 스트리밍한다.
* 사이드카 컨테이너는 로깅 에이전트를 실행하며, 애플리케이션 컨테이너에서 로그를 가져오도록 구성된다.

#### 사이드카 컨테이너 스트리밍

![스트리밍 컨테이너가 있는 사이드카 컨테이너](/images/docs/user-guide/logging/logging-with-streaming-sidecar.png)

사이드카 컨테이너가 자체 `stdout` 및 `stderr` 스트림으로
쓰도록 하면, 각 노드에서 이미 실행 중인 kubelet과 로깅 에이전트를
활용할 수 있다. 사이드카 컨테이너는 파일, 소켓 또는 journald에서 로그를 읽는다.
각 사이드카 컨테이너는 자체 `stdout` 또는 `stderr` 스트림에 로그를 출력한다.

이 방법을 사용하면 애플리케이션의 다른 부분에서 여러 로그 스트림을
분리할 수 ​​있고, 이 중 일부는 `stdout` 또는 `stderr` 에
작성하기 위한 지원이 부족할 수 있다. 로그를 리디렉션하는 로직은
최소화되어 있기 때문에, 심각한 오버헤드가 아니다. 또한,
`stdout` 및 `stderr` 가 kubelet에서 처리되므로, `kubectl logs` 와 같은
빌트인 도구를 사용할 수 있다.

예를 들어, 파드는 단일 컨테이너를 실행하고, 컨테이너는
서로 다른 두 가지 형식을 사용하여 서로 다른 두 개의 로그 파일에 기록한다. 파드에 대한
구성 파일은 다음과 같다.

{{< codenew file="admin/logging/two-files-counter-pod.yaml" >}}

두 컴포넌트를 컨테이너의 `stdout` 스트림으로 리디렉션한 경우에도, 동일한 로그
스트림에 서로 다른 형식의 로그 항목을 작성하는 것은
추천하지 않는다. 대신, 두 개의 사이드카 컨테이너를 생성할 수 있다. 각 사이드카
컨테이너는 공유 볼륨에서 특정 로그 파일을 테일(tail)한 다음 로그를
자체 `stdout` 스트림으로 리디렉션할 수 있다.

다음은 사이드카 컨테이너가 두 개인 파드에 대한 구성 파일이다.

{{< codenew file="admin/logging/two-files-counter-pod-streaming-sidecar.yaml" >}}

이제 이 파드를 실행하면, 다음의 명령을 실행하여 각 로그 스트림에
개별적으로 접근할 수 있다.

```shell
kubectl logs counter count-log-1
```

출력은 다음과 같다.

```console
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```

```shell
kubectl logs counter count-log-2
```

출력은 다음과 같다.

```console
Mon Jan  1 00:00:00 UTC 2001 INFO 0
Mon Jan  1 00:00:01 UTC 2001 INFO 1
Mon Jan  1 00:00:02 UTC 2001 INFO 2
...
```

클러스터에 설치된 노드-레벨 에이전트는 추가 구성없이
자동으로 해당 로그 스트림을 선택한다. 원한다면, 소스 컨테이너에
따라 로그 라인을 파싱(parse)하도록 에이전트를 구성할 수 있다.

참고로, CPU 및 메모리 사용량이 낮음에도 불구하고(cpu에 대한 몇 밀리코어의
요구와 메모리에 대한 몇 메가바이트의 요구), 로그를 파일에 기록한 다음
`stdout` 으로 스트리밍하면 디스크 사용량은 두 배가 될 수 있다. 단일 파일에
쓰는 애플리케이션이 있는 경우, 일반적으로 스트리밍
사이드카 컨테이너 방식을 구현하는 대신 `/dev/stdout` 을 대상으로
설정하는 것을 추천한다.

사이드카 컨테이너를 사용하여 애플리케이션 자체에서 로테이션할 수 없는
로그 파일을 로테이션할 수도 있다. 이 방법의 예시는 정기적으로 `logrotate` 를 실행하는 작은 컨테이너를 두는 것이다.
그러나, `stdout` 및 `stderr` 을 직접 사용하고 로테이션과
유지 정책을 kubelet에 두는 것이 권장된다.

#### 로깅 에이전트가 있는 사이드카 컨테이너

![로깅 에이전트가 있는 사이드카 컨테이너](/images/docs/user-guide/logging/logging-with-sidecar-agent.png)

노드-레벨 로깅 에이전트가 상황에 맞게 충분히 유연하지 않은 경우,
애플리케이션과 함께 실행하도록 특별히 구성된 별도의 로깅 에이전트를 사용하여
사이드카 컨테이너를 생성할 수 있다.

{{< note >}}
사이드카 컨테이너에서 로깅 에이전트를 사용하면
상당한 리소스 소비로 이어질 수 있다. 게다가, kubelet에 의해
제어되지 않기 때문에, `kubectl logs` 를 사용하여 해당 로그에
접근할 수 없다.
{{< /note >}}

여기에 로깅 에이전트가 포함된 사이드카 컨테이너를 구현하는 데 사용할 수 있는 두 가지 구성 파일이 있다. 첫 번째 파일에는
fluentd를 구성하기 위한 [`ConfigMap`](/docs/tasks/configure-pod-container/configure-pod-configmap/)이 포함되어 있다.

{{< codenew file="admin/logging/fluentd-sidecar-config.yaml" >}}

{{< note >}}
fluentd를 구성하는 것에 대한 자세한 내용은, [fluentd 문서](https://docs.fluentd.org/)를 참고한다.
{{< /note >}}

두 번째 파일은 fluentd가 실행되는 사이드카 컨테이너가 있는 파드를 설명한다.
파드는 fluentd가 구성 데이터를 가져올 수 있는 볼륨을 마운트한다.

{{< codenew file="admin/logging/two-files-counter-pod-agent-sidecar.yaml" >}}

이 예시 구성에서, 사용자는 애플리케이션 컨테이너 내의 모든 소스을 읽는 fluentd를 다른 로깅 에이전트로 대체할 수 있다.

### 애플리케이션에서 직접 로그 노출

![애플리케이션에서 직접 로그 노출](/images/docs/user-guide/logging/logging-from-application.png)

모든 애플리케이션에서 직접 로그를 노출하거나 푸시하는 클러스터-로깅은 쿠버네티스의 범위를 벗어난다.
