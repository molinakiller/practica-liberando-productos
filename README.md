# practica-liberando-productos

### Endpoint bye bye and configuration of alerts

For deploy a the Bye endpoint we must add these code in python on /src/application/app.py file.

```python
    @app.get("/bye")
    async def bye_endpoint():
        """Implement health check endpoint"""
        #┬áIncrement counter used for register the total number of calls in the webserver
        REQUESTS.inc()
        # Increment counter used for register the requests to bye endpoint
        BYE_ENDPOINT_REQUESTS.inc()
        return {"msg": "Bye Bye"} 
```

Then we have to create the counter for Prometheus, adding this code line on the top of file.
This line its for say to prometheus that it must take a counter every time that endpoint were visited.

```python
BYE_ENDPOINT_REQUESTS = Counter('bye_requests_total', 'Total number of requests to main endpoint')
```

### Unit test for endpoint bye bye

We must create the unit test for the new bye endpoint. We must add these code in python on /src/tests/app_test.py file.
Just for be sure that when we visit /bye endpoint it response '{"msg": "Bye Bye"}'

```python
    @pytest.mark.asyncio
    async def bye_main_test(self):
        """Tests the bye endpoint"""
        response = client.get("bye")

        assert response.status_code == 200
        assert response.json() == {"msg": "Bye Bye"}
```

### CI/CD with Github Actions

I choose Github Action becouse i dont have enougth experiencie with it. For deploy it, we must create a repo, and then go to Actions tabs.

#### Testing workflow

When you click on Actions tab, you can start your custom Workflow or strart with a template.
I create a simple workflow, for every push on main branch with the followings steps:
1) Setup python 3.10 version
2) Install dependencies
3) Check sintax and programming errors with the library Flake8
4) Do the test with coverage

```yaml
name: Python application

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

permissions:
  contents: read

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Set up Python 3.10
      uses: actions/setup-python@v3
      with:
        python-version: "3.10"
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install flake8 pytest
        if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
    - name: Lint with flake8
      run: |
        # stop the build if there are Python syntax errors or undefined names
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        # exit-zero treats all errors as warnings. The GitHub editor is 127 chars wide
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics
    - name: Test with pytest
      run: |
        pytest --cov --cov-report=html 
```

#### Docker push workflow
First of all you need to create a personal acces token that have a workflow permision. 
I create a simple workflow, for every push on any branch with the followings steps:
1) Unshallow
2) Get the version to publish
3) Set up Docker Buildx
4) Docker Login in GHCR (git registry)
5) Build and push Docker image

```yaml
name: docker
on:
  pull_request:
    branches:
      - '*'
  push:
    tags:
      - 'v*' 
    branches:
      - '*'    
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Unshallow
        run: git fetch --prune --unshallow

      - name: Get the version to publish
        id: get_version
        run: echo ::set-output name=VERSION::${GITHUB_REF#refs/tags/v}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Docker Login in GHCR (git registry)
        uses: docker/login-action@v1
        id: configure-login-ghcr
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.TOKEN }}

      - name: Build and push Docker image
        run: make publish
        env:
          VERSION: ${{ steps.get_version.outputs.VERSION }}
```
The final step its push the tag:

```bash
git tag -a v0.0.1 -m "publish version v0.0.1"
git push origin --tags
```

Every time we increase the version, we must change in the /python-monitoring/values.yaml file.

### Create Slack chanel and configure Webhook

First of all you will need login in Slack, create a chanel and then install the App "Incoming Webhoks"
Then you must link your chanel with the webhook, this step will create a URL webhook.
With this URL you must configure the file "custom_values_prometheus.yaml" adding the Webhook and your chanel name.

### Deploy Prometheus and configure alerts

#### Requirements

We will deploy a prometheus and grafana stack to monitoring the web app in python.

1) [Minikube](https://minikube.sigs.k8s.io/docs/start/)
2) [Kubectl](https://kubernetes.io/docs/tasks/tools/)
3) [heml](https://helm.sh/docs/intro/install/)

#### Steps
1) Create the Kubernetes kluster using the following command of minikube with 4GB of memory:

```bash
minikube start --kubernetes-version='v1.21.1' \
    --memory=4096 \
    -p monitoring-demo
```
2) Add helm repo prometheus-community, to deploy the kube-prometheus-stack chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

3) Deploy the kube-prometheus-stack chart trought main repository of helm. We will use the /monitoring/custom_values_prometheus.yaml  file from this repository.
Firts of all you must add the webhook of slack that you add in the last step, and put in the line 109.
```bash
cd monitoring
helm -n monitoring upgrade --install prometheus prometheus-community/kube-prometheus-stack -f custom_values_prometheus.yaml --create-namespace --wait --version 34.1.1
```
To see the pods created: 
```bash
kubectl --namespace monitoring get pods -l "release=prometheus"
```

4) Install metrics-server in minikube trought addon activation:

```
minikube addons enable metrics-server -p monitoring-demo
```
5) Install python app helm

```bash
cd ..
helm -n fast-api upgrade my-app --install --create-namespace python-monitoring/
```
It must show the following
```bash
Release "my-app" does not exist. Installing it now.
NAME: my-app
LAST DEPLOYED: Mon Apr 25 19:59:58 2022
NAMESPACE: fast-api
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace fast-api -l "app.kubernetes.io/name=fast-api-webapp,app.kubernetes.io/instance=my-app" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace fast-api $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace fast-api port-forward $POD_NAME 8080:$CONTAINER_PORT
```
To get prometheus web interface, we must do a portforward
```bash
kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090
```

To delete the helm
```bash
helm list -n monitoring
helm uninstall my-app -n fast-api
```

### Alert manager configuration

For configure Alert manager we must go to monitoring and add the chanel and webhook.
When we did all the steps slack must advise us like that:

```bash
[FIRING:2] Monitoring Event Notification
Alert: PrometheusRuleFailures - critical
 Description:
 Graph: :gr├ífico_con_tendencia_ascendente: Runbook: :cuaderno_de_espiral:
 Details:
  ÔÇó alertname: PrometheusRuleFailures
  ÔÇó container: prometheus
  ÔÇó endpoint: http-web
  ÔÇó instance: 172.17.0.7:9090
  ÔÇó job: prometheus-kube-prometheus-prometheus
  ÔÇó namespace: monitoring
  ÔÇó pod: prometheus-prometheus-kube-prometheus-prometheus-0
  ÔÇó prometheus: monitoring/prometheus-kube-prometheus-prometheus
  ÔÇó rule_group: /etc/prometheus/rules/prometheus-prometheus-kube-prometheus-prometheus-rulefiles-0/monitoring-prometheus-kube-prometheus-kubelet.rules-44f1056b-cf89-41c5-9e75-2bcc767b4d26.yaml;kubelet.rules
  ÔÇó service: prometheus-kube-prometheus-prometheus
  ÔÇó severity: critical
 
 Alert: PrometheusRuleFailures - critical
 Description:
 Graph: :gr├ífico_con_tendencia_ascendente: Runbook: :cuaderno_de_espiral:
 Details:
  ÔÇó alertname: PrometheusRuleFailures
  ÔÇó container: prometheus
  ÔÇó endpoint: http-web
  ÔÇó instance: 172.17.0.7:9090
  ÔÇó job: prometheus-kube-prometheus-prometheus
  ÔÇó namespace: monitoring
  ÔÇó pod: prometheus-prometheus-kube-prometheus-prometheus-0
  ÔÇó prometheus: monitoring/prometheus-kube-prometheus-prometheus
  ÔÇó rule_group: /etc/prometheus/rules/prometheus-prometheus-kube-prometheus-prometheus-rulefiles-0/monitoring-prometheus-kube-prometheus-kubernetes-system-kubelet-639735c3-303e-43c3-af8a-8e5e243eb32c.yaml;kubernetes-system-kubelet
  ÔÇó service: prometheus-kube-prometheus-prometheus
  ÔÇó severity: critical

  ---

  [RESOLVED] Monitoring Event Notification
  Alert: InfoInhibitor - none
 Description:
 Graph: :gr├ífico_con_tendencia_ascendente: Runbook: :cuaderno_de_espiral:
 Details:
  ÔÇó alertname: InfoInhibitor
  ÔÇó alertstate: pending
  ÔÇó container: config-reloader
  ÔÇó namespace: monitoring
  ÔÇó pod: prometheus-prometheus-kube-prometheus-prometheus-0
  ÔÇó prometheus: monitoring/prometheus-kube-prometheus-prometheus
  ÔÇó severity: none
```
### Stress test 

We must login to the python deployment, we could use this command:

```bash
export POD_NAME=$(kubectl get pods --namespace fast-api -l "app.kubernetes.io/name=fast-api-webapp,app.kubernetes.io/instance=my-app" -o jsonpath="{.items[0].metadata.name}")
kubectl -n fast-api exec --stdin --tty $POD_NAME -- /bin/sh
```
OR
```bash
kubectl get all -n fast-api
kubectl exec -it POD-NAME -n fast-api /bin/sh
```

Then we must install a github project that contains the estress funcionality, git the followings command:
```bash
apk update && apk add git go
git clone https://github.com/jaeg/NodeWrecker.git
cd NodeWrecker
go build -o extress main.go
./extress -abuse-memory -escalate -max-duration 10000000
```


### Grafana dashboard

To login on grafana, we must do a portforward
```bash
kubectl -n monitoring port-forward svc/prometheus-grafana 3000:80
```
We must go to http://localhost:3000 on browser, the credentials by default its admin for user and prom-operator for password.

Then you have to download or copy the custom-dashboard-grafana.yaml file of repo.

In the left menu you must choose the (+) > Import > Import from json.

When you import properly the dashboard you must going to settings of dashboard,
then add the variables that we are using on the custom file.

Dashboard Settings  > Variables > (it will be 2 variables) You must click on each and disable the following:

1) Multi-value
2) Include All option

It will be show something like this
![Image text](https://github.com/molinakiller/practica-liberando-productos/blob/main/grafana.png)
