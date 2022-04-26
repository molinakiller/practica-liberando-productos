# practica-liberando-productos

### Endpoint bye bye and configuration of alerts

For deploy a the Bye endpoint we must add these code in python on /src/application/app.py file.

```python
    @app.get("/bye")
    async def bye_endpoint():
        """Implement health check endpoint"""
        # Increment counter used for register the total number of calls in the webserver
        REQUESTS.inc()
        # Increment counter used for register the requests to healtcheck endpoint
        BYE_ENDPOINT_REQUESTS.inc()
        return {"msg": "Bye Bye"} 
```

Then we have to create the counter for Prometheus, adding this code line.

```python
BYE_ENDPOINT_REQUESTS = Counter('bye_requests_total', 'Total number of requests to main endpoint')
```

### Unit test for endpoint bye bye

We must create the unit test for the new endpoint. We must add these code in python on /src/tests/app_test.py file.

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

```
git tag -a v0.0.1 -m "publish version v0.0.1"
git push origin --tags
```

### Create Slack chanel and configure Webhook

First of all you will need login in Slack, create a chanel and then install the App "Incoming Webhoks"
Then you must link your chanel with the webhook, this step will create a URL webhook.
With this URL you must configure the file "custom_values_prometheus.yaml" adding the Webhook and your chanel name.

### Deploy Prometheus and configure alerts

#### Requirements

1) Minikube
2) Kubectl
3) heml

#### Steps
1) Create the Kubernetes kluster using the following command of minikube:

```
minikube start --kubernetes-version='v1.21.1' \
    --memory=4096 \
    -p monitoring-demo
```
2) Add helm repo prometheus-community to deploy the chart kube-prometheus-stack:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

3) Desplegar el chart kube-prometheus-stack del repositorio de helm añadido en el paso anterior con los valores configurados en el archivo custom_values_prometheus.yaml en el namespace monitoring:
```
cd monitoring
helm -n monitoring upgrade --install prometheus prometheus-community/kube-prometheus-stack -f custom_values_prometheus.yaml --create-namespace --wait --version 34.1.1
```
4) Install metrics-server in minikube trought addon activation:

```
minikube addons enable metrics-server -p monitoring-demo
```
5) Install python app helm

```
cd ..
helm -n fast-api upgrade my-app --install --create-namespace python-monitoring/
```
It must show the following
```
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
To delete the helm
```
helm uninstall my-app -n fast-api
```

### Alert manager configuration

### Grafana dashboard