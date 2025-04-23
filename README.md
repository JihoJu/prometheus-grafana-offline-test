# 로컬 Minikube 환경 Prometheus/Grafana 오프라인 테스트

이 저장소는 로컬 Minikube 환경에서 Prometheus 및 Grafana 모니터링 스택을 설치하고, 오프라인(폐쇄망) 환경을 시뮬레이션하여 간단한 샘플 애플리케이션을 모니터링하는 과정을 테스트하고 기록하기 위한 것입니다.

## 목적

*   폐쇄망 환경을 가정하여 Prometheus/Grafana 스택의 기본 설치 및 동작 검증
*   Helm 차트 및 커스텀 values 파일을 이용한 구성 관리
*   샘플 애플리케이션의 메트릭을 Prometheus가 수집하고 Grafana 대시보드에 시각화하는 과정 확인

## 사전 요구사항

*   [Minikube](https://minikube.sigs.k8s.io/docs/start/) (v1.35.0 기준 테스트됨)
*   [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
*   [Helm](https://helm.sh/docs/intro/install/) (v3+)
*   [Docker](https://docs.docker.com/get-docker/) (이미지 다운로드 및 Minikube 로딩용)

## 필수 아티팩트  repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    helm fetch prometheus-community/kube-prometheus-stack --version 45.0.0
    # 결과: kube-prometheus-stack-45.0.0.tgz 파일 생성됨
    ```

**2. Docker Images:** (아래는 예시이며, 사용하는 차트 버전에 따라 다를 수 있음)
*   `quay.io/prometheus-operator/prometheus-operator:v0.63.0`
*   `quay.io/prometheus/prometheus:v2.43.0`
*   `quay.io/prometheus/alertmanager:v0.25.0`
*   `grafana/grafana:9.4.7`
*   `registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.8.2`
*   `quay.io/prometheus/node-exporter:v1.5.0`
*   `quay.io/prometheus-operator/prometheus-config-reloader:v0.63.0`
*   `jimmidyson/configmap-reload:v0.5.0`
*   `prom/prometheus-dummy-exporter:v0.2.0` (샘플 앱용)
    ```bash
    # 예시: docker pull quay.io/prometheus-operator/prometheus-operator:v0.63.0
    # ... 목록의 모든 이미지를 docker pull 수행 ...
    ```

## 설정 및 테스트 절차

**(이제 인터넷 연결을 끊거나 오프라인 환경이라고 가정합니다)**

1.  **Minikube 이미지 로드:** 다운로드한 모든 Docker 이미지를 Minikube 내부로 로드합니다.
    ```bash
    # 예시: minikube image load quay.io/prometheus-operator/prometheus-operator:v0.63.0
    # ... 모든 필수 이미지에 대해 실행 ...
    minikube image load prom/prometheus-dummy-exporter:v0.2.0
    ```

2.  **네임스페이스 생성:** 모니터링 스택을 위한 네임스페이스를 생성합니다.
    ```bash
    kubectl create namespace monitoring
    ```

3.  **Prometheus 스택 설치:** 다운로드한 Helm 차트와 제공된 `helm/minikube-values.yaml` 파일을 사용하여 설치합니다.
    ```bash
    helm install prometheus-stack ./kube-prometheus-stack-45.0.0.tgz \
      --namespace monitoring \
      -f helm/minikube-values.yaml
    # Pod들이 Running 상태가 될 때까지 기다립니다.
    kubectl get pods -n monitoring -w
    ```

4.  **샘플 애플리케이션 배포:** `manifests/sample-app/` 디렉토리의 매니페스트를 적용합니다.
    ```bash
    kubectl apply -f manifests/sample-app/
    # Pod 및 Service 생성 확인
    kubectl get pods,svc -n default # sample-app이 배포된 네임스페이스 확인
    ```

5.  **Prometheus UI 확인:**
    ```bash
    kubectl port-forward -n monitoring svc/prometheus-operated 9090
    # 브라우저에서 http://localhost:9090 접속
    # Status > Targets 에서 sample-app 관련 타겟이 UP 상태인지 확인
    # Graph 탭에서 'dummy_metric' 쿼리 실행
    ```

6.  **Grafana UI 확인:**
    ```bash
    minikube service -n monitoring prometheus-stack-grafana --url
    # 출력된 URL로 브라우저에서 접속
    # 로그인: admin / prom-operator (minikube-values.yaml에 정의된 비밀번호)
    # Configuration > Data Sources 에서 Prometheus 연결 확인
    # + > Dashboard > Add new panel 에서 'dummy_metric' 쿼리로 데이터 시각화 확인
    ```

## 정리

테스트 완료 후 리소스를 정리합니다.
```bash
kubectl delete -f manifests/sample-app/
helm uninstall prometheus-stack -n monitoring
kubectl delete namespace monitoring
# 필요시: minikube stop 또는 minikube delete오프라인 전 다운로드 필요)

