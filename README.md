# deploy Partial Helm Chart

## Background
  - Sometimes you need to install/upgrade only part of the resources of helm chart or only one sub-chart,
    but you don't want to modify the helm chart or values.yaml ,or can't do it(for example , if you don't want to rollout rest of resources in production and your helm chart doesn't know how to handle  rolling update of deployments only for ones that a configuration or image actually changed) 
  - Off course you can always install/upgrade only one sub-chart, but this enforce you to manage another helm chart , and you need to keep track of another values.yaml and migrate all portion of sub-chart in values.yaml of parent chart into this values yaml, but let assume that this is acceptable,
    You still have the following scenario in the next bullet.
  - You want to deploy only part of the chart(for example , if you have two sets of {deployments,service,configmap, secret} in the same chart), Then you must need to find a way how to install or upgrade only part of the resources.
    
## Solution

1. Using helm template/helm install/upgrade --dry-run option, to build k8 manifests and persist to a file manifests.yaml .
2. Label only desired resources, filter by component name or sub-chart name.
   Labeling can be done using the following techniques:
    - Manually labeling all desired resources in manifests.yaml
    - Use `kustomize` tool to automatically label all resources that their name following a pattern that you're specifying(recommended - easy to use), we will use kustomiz'e SMP(Strategic merge patch).
    - Use scripting tools like `sed` to label resources(more complicated - less recommended) 
3. After labeling desired resources with a well known label and value in manifests.yaml file, perform oc apply  -f -l labelname=value on all manifestst.

### Implementation

Our reference helm chart for example is an umbrella chart with 4 microservices as sub-charts
```shell
demo-chart/
├── charts
│   ├── microservice1
│   │   ├── charts
│   │   ├── Chart.yaml
│   │   ├── templates
│   │   │   ├── deployment.yaml
│   │   │   ├── _helpers.tpl
│   │   │   ├── hpa.yaml
│   │   │   ├── ingress.yaml
│   │   │   ├── NOTES.txt
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── service.yaml
│   │   │   └── tests
│   │   │       └── test-connection.yaml
│   │   └── values.yaml
│   ├── microservice2
│   │   ├── charts
│   │   ├── Chart.yaml
│   │   ├── templates
│   │   │   ├── deployment.yaml
│   │   │   ├── _helpers.tpl
│   │   │   ├── hpa.yaml
│   │   │   ├── ingress.yaml
│   │   │   ├── NOTES.txt
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── service.yaml
│   │   │   └── tests
│   │   │       └── test-connection.yaml
│   │   └── values.yaml
│   ├── microservice3
│   │   ├── charts
│   │   ├── Chart.yaml
│   │   ├── templates
│   │   │   ├── deployment.yaml
│   │   │   ├── _helpers.tpl
│   │   │   ├── hpa.yaml
│   │   │   ├── ingress.yaml
│   │   │   ├── NOTES.txt
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── service.yaml
│   │   │   └── tests
│   │   │       └── test-connection.yaml
│   │   └── values.yaml
│   └── microservice4
│       ├── charts
│       ├── Chart.yaml
│       ├── templates
│       │   ├── deployment.yaml
│       │   ├── _helpers.tpl
│       │   ├── hpa.yaml
│       │   ├── ingress.yaml
│       │   ├── NOTES.txt
│       │   ├── serviceaccount.yaml
│       │   ├── service.yaml
│       │   └── tests
│       │       └── test-connection.yaml
│       └── values.yaml
├── Chart.yaml
├── kustomize
│   ├── kustomization-base.yaml
│   ├── kustomization.yaml
│   ├── manifests.yaml
│   └── patch.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```
#### Using Kustomize:

1. Create manifests.yaml containing all manifests of all chart and sub-charts using helm template:
```shell
[zgrinber@zgrinber deploy-partial-helm-chart]$ helm template demo-chart demo-chart/ > demo-chart/kustomize/manifests.yaml
```

2. Go to this repository' root folder or make sure you're there.
3. Create patch that will contain the label and its value
```shell
cat > ./demo-chart/kustomize/patch.yaml << EOF
apiVersion: v1
kind: all
metadata:
  name: nevermind
  labels:
    adhocInstall: "true"
EOF
```
3. Create kustomization.yaml file
```shell
cat > ./demo-chart/kustomize/kustomization-base.yaml << EOF
resources:
- manifests.yaml

patches:
  - path: patch.yaml  
    target:
      name: "ms_name.*"
EOF
```

4. Assuming that we want to install only microservice3 (out of 4 microservice), then we need to run the following command:
```shell
sed 's/name: "ms_name.*"/name: "microservice3.*"/' demo-chart/kustomize/kustomization-base.yaml  | tee demo-chart/kustomize/kustomization.yaml | oc kustomize ./demo-chart/kustomize/ | oc apply -f - -l adhocInstall=true --dry-run=client -o yaml
serviceaccount/microservice3 created
service/microservice3 created
deployment.apps/microservice3 created
pod/microservice3-test-connection created
```

#### Using sed:
1. Go to this repository' root folder or make sure you're there.
2. Assuming that you want to deploy only microservice4 (out of 4 microservices) then:
```shell
export MS_NAME=microservice4;helm template demo-chart demo-chart/ | tr '\n' ";" | sed -E 's/[[:blank:]]{2}name:\s{1,2}'$MS_NAME'[a-zA-Z0-9\-]*;(  namespace: [a-zA-Z0-9\-]+;){0,1}\s{2}labels:\s*;/&    adhocInstall: "true";/g' | sed -E 's/[[:blank:]]{4}helm\.sh\/chart: '$MS_NAME'[-.a-zA-Z0-9]*;/&    adhocInstall: "true";/g' | tr ';' '\n' | oc apply -f - -l adhocInstall=true
serviceaccount/microservice4 created
service/microservice4 created
deployment.apps/microservice4 created
pod/microservice4-test-connection created

```