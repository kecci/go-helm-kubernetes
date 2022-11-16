# go-helm-kubernetes

There are several steps to running with helm:
- [go-helm-kubernetes](#go-helm-kubernetes)
  - [Build image with docker](#build-image-with-docker)
  - [Install helm](#install-helm)
  - [Create a new charts](#create-a-new-charts)
  - [Setup Config](#setup-config)
  - [Check Helm Template](#check-helm-template)
  - [Deploy helm](#deploy-helm)
  - [Get list of pods, and svc](#get-list-of-pods-and-svc)
  - [Source](#source)

## Build image with docker
Before we deploy with helm. We should prepare the image and we build with `Dockerfile` and run this command:

`docker build -t kecci/go-helm-kubernetes .`

## Install helm
`brew install helm`

If done, check the version

`helm version`
```
version.BuildInfo{Version:"v3.10.2", GitCommit:"50f003e5ee8704ec937a756c646870227d7c8b58", GitTreeState:"clean", GoVersion:"go1.19.3"}
```

## Create a new charts
`helm create charts/go-helm-kubernetes`
```
Creating charts/go-helm-kubernetes
```

## Setup Config

1. Remove unused yaml. inside of `charts/go-helm-kubernetes` :
```
/charts
/templates/test
/templates/hpa.yaml
/templates/ingress.yaml
/templates/serviceaccount.yaml
```

2. Open `values.yaml`

let's focus on this line, and update the values to the name of image:
```yaml
replicaCount: 2 # how many replicas

image:
  repository: kecci/go-helm-kubernetes # name of your image on docker
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "latest"

  ...

service:
  type: NodePort # changes to NodePort
  port: 80 # port of your app running
```

2. Open `Chart.yaml`

update the values for the version of image:
```yaml
appVersion: "latest"
```

3. Open the `deployment.yaml`

Remove unused:
- imagePullSecrets
- serviceAccountName
- securityContext
- livelinessProbe
- readinessProbe
- resources
- nodeSelector
- affinity
- tolerations

until like this:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "go-helm-kubernetes.fullname" . }}
  labels:
    {{- include "go-helm-kubernetes.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "go-helm-kubernetes.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "go-helm-kubernetes.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
```

## Check Helm Template
`helm template ./charts/go-helm-kubernetes/`
```yaml
---
# Source: go-helm-kubernetes/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: release-name-go-helm-kubernetes
  labels:
    helm.sh/chart: go-helm-kubernetes-0.1.0
    app.kubernetes.io/name: go-helm-kubernetes
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "latest"
    app.kubernetes.io/managed-by: Helm
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: go-helm-kubernetes
    app.kubernetes.io/instance: release-name
---
# Source: go-helm-kubernetes/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: release-name-go-helm-kubernetes
  labels:
    helm.sh/chart: go-helm-kubernetes-0.1.0
    app.kubernetes.io/name: go-helm-kubernetes
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "latest"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: go-helm-kubernetes
      app.kubernetes.io/instance: release-name
  template:
    metadata:
      labels:
        app.kubernetes.io/name: go-helm-kubernetes
        app.kubernetes.io/instance: release-name
    spec:
      containers:
        - name: go-helm-kubernetes
          image: "kecci/go-helm-kubernetes:latest"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```

## Deploy helm

`helm install go-helm-kubernetes ./charts/go-helm-kubernetes`

```
NAME: go-helm-kubernetes
LAST DEPLOYED: Wed Nov 16 18:11:30 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services go-helm-kubernetes)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
```

## Get list of pods, and svc
`kubectl get po,svc`
```
NAME                                     READY   STATUS    RESTARTS   AGE
pod/go-helm-kubernetes-c9c9dfbd8-4xc7p   1/1     Running   0          33s
pod/go-helm-kubernetes-c9c9dfbd8-qwwfj   1/1     Running   0          33s

NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/go-helm-kubernetes   NodePort    10.109.88.209   <none>        80:30446/TCP   33s
service/kubernetes           ClusterIP   10.96.0.1       <none>        443/TCP        2d5h
```

Open web: `http://localhost:30446`

## Source
- https://www.youtube.com/watch?v=H6pF2Swqrko&t=7800s
- https://frankie-yanfeng.github.io/2019/10/15/Helm-2019/