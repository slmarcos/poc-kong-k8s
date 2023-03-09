# POC - Kong API Gateway com Kubernetes
---

## Ferramentas necessárias

- [Kind](https://kind.sigs.k8s.io/)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm 3](https://helm.sh/)

## Detalhes do projeto

    .
    ├── apps
    │   └── users
    │       ├── deployment.yaml
    │       ├── hpa.yaml
    │       ├── ingress.yaml
    │       ├── kong-ingress.yaml
    │       └── service.yaml
    ├── kind
    │   └── kind-config.yaml
    ├── kong
    │   ├── kong-config.yaml
    │   └── plugins
    │       └── kong-prometheus.yaml
    ├── prometheus
    │   └── prometheus.yaml
    └── README.md

Na pasta `kind` temos o yaml com as configurações necessárias para criação do cluster localmente.

Na pasta `kong`temos na raiz a o arquivo com as configurações para instalação do `Kong Gateway`. E na pasta `plugins`temos o arquivo do plugin do `Prometheus` usado para coletar métricas do cluster.

Na raiz temos a pasta `prometheus` com um yaml de configurações que devem ser passadas na instalação do `Prometheus`.

Na pasta `apps` temos os arquivos `Kubernetes` necessários para subir uma aplicação usando a imagem do `echo server` para que possamos fazer os testes.

## Setup do ambiente

Após o ambiente configurado e iniciado corretamente será possível realizar os testes enviando requisições para o endereço http://localhost/users. O `echo server` retorna os dados do pod que processou a requisição e informações da requisição.

Os seguintes comandos devem ser usados para execução do projeto:

### Criação do cluster:
```console
$ kind create cluster --config ./kind/kind-config.yaml
```
### Instalação do Kong

```console
$ helm install kong kong/kong -f ./kong/kong-config.yaml \
  --set proxy.type=NodePort,proxy.http.nodePort=30000,proxy.tls.nodePort=30003 \
  --set ingressController.installCRDs=false \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.labels.release=promstack \
  -n kong --create-namespace
```


### Instalação do metric server

```console
$ helm upgrade --install metrics-server metrics-server/metrics-server --namespace kube-system --set args={--kubelet-insecure-tls}
```

### Instalação do Prometheus

```console
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ kubectl create ns monitoring
$ helm install prometheus-stack prometheus-community/kube-prometheus-stack -f prometheus/prometheus.yaml --namespace monitoring
```

### Instalação do Plugin Kong Prometheus

```console
$ kubectl apply -f ./kong/plugins
```

### Instalação da aplicação
```console
$ kubectl create ns users
$ kubectl apply -f ./apps/users -n users
```

### Visualização das métricas

Para visualização das métricas é necessários rodar o seguinte comando no terminal para que o Grafana fique acessível pelo host local.

```console
$ kubectl port-forward -n kong svc/prometheus-stack-grafana 3000:80 -n monitoring
```


O dados para acesso são:

```
url: http://localhost:3000
usuário: admin
senha: prom-operator
```


Após feito login podemos importar o dashboard do Kong para o Grafana.
Em dashboard clique em:

```
New -> import -> coloque o ID 7424 -> Load -> no ultimo campo selecione Prometheus -> Import
```

Após alguns segundos já será possível visualizar as métricas em tempo real.

Caso não receba as métricas rode o comando abaixo para verificar se o `service monitor` está rodando para o `namespace` do `Kong`.

```console
$ kubectl get servicemonitors -n kong
```
O seguinte resultado deve ser exibido:
```console
NAME        AGE
kong-kong   9s
```
Caso não apareça remova o `Kong` e instale novamente.
