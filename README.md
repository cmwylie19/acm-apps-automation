# Automate Application Deployment in ACM
_These guidelines assume that you are running RHACM 2.3, and that you have connected two managed clusters to the HUB._

*As a prerequisite, you will need helm 3.*

Outlined below are the expected labels that your clusters must contains in order to the Placement API to work as intended for this demo.

West Managed Cluster:
```
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  labels:
    cloud: Amazon
    env: dev # DEV
    name: serverless-cluster
    openshiftVersion: 4.4.29
    region: us-west-1 # WEST 
    vendor: OpenShift
  name: serverless-cluster
spec:
  hubAcceptsClient: true
  leaseDurationSeconds: 60
```

East Managed Cluster:
```
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  labels:
    cloud: Amazon
    env: dev # DEV
    name: sm-cluster
    openshiftVersion: 4.7.16
    region: us-east-1 # EAST
    vendor: OpenShift
  name: sm-cluster
spec:
  hubAcceptsClient: true
  leaseDurationSeconds: 60
```

## Demo
In your HUB cluster, install the helm chart, for clarity, give it a name of `deploy-apps`.

The outline to install a helm chart is as follows:   
`helm install [name the release] [chart]`

Assuming you have downloaded this repository, and are in the base directory.
```
helm install deploy-apps deploy-apps
```

output
```
NAME: deploy-apps
LAST DEPLOYED: Wed Oct  6 19:40:29 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

If you are curios about what was deployed, run the command below to get the output.
```
helm get all deploy-apps
```

output
```
NAME: deploy-apps
LAST DEPLOYED: Wed Oct  6 19:40:29 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
null

COMPUTED VALUES:
affinity: {}
autoscaling:
  enabled: false
  maxReplicas: 100
  minReplicas: 1
  targetCPUUtilizationPercentage: 80
fullnameOverride: ""
image:
  pullPolicy: IfNotPresent
  repository: nginx
  tag: ""
imagePullSecrets: []
ingress:
  annotations: {}
  className: ""
  enabled: false
  hosts:
  - host: chart-example.local
    paths:
    - path: /
      pathType: ImplementationSpecific
  tls: []
nameOverride: ""
nodeSelector: {}
podAnnotations: {}
podSecurityContext: {}
replicaCount: 1
resources: {}
securityContext: {}
service:
  port: 80
  type: ClusterIP
serviceAccount:
  annotations: {}
  create: true
  name: ""
tolerations: []

HOOKS:
MANIFEST:
---
# Source: application-deploy/templates/bookinfo-app.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bookinfo
---
# Source: application-deploy/templates/bookinfo-app.yaml
apiVersion: app.k8s.io/v1beta1
kind: Application
metadata:
  name: bookinfo-app
  namespace: bookinfo
spec:
  componentKinds:
  - group: apps.open-cluster-management.io
    kind: Subscription
  descriptor: {}
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - bookinfo-app
---
# Source: application-deploy/templates/httpbin-app.yaml
apiVersion: app.k8s.io/v1beta1
kind: Application
metadata:
  name: httpbin-app
  namespace: default
spec:
  componentKinds:
  - group: apps.open-cluster-management.io
    kind: Subscription
  descriptor: {}
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - httpbin-app
---
# Source: application-deploy/templates/bookinfo-app.yaml
apiVersion: apps.open-cluster-management.io/v1
kind: Channel
metadata:
  name: bookinfo-app-latest
  namespace: bookinfo
spec:
  type: GitHub
  pathname: https://github.com/cmwylie19/bookinfo-acm.git
---
# Source: application-deploy/templates/httpbin-app.yaml
apiVersion: apps.open-cluster-management.io/v1
kind: Channel
metadata:
  name: httpbin-app-latest
  namespace: default
spec:
  type: GitHub
  pathname: https://github.com/cmwylie19/bookinfo-acm.git
---
# Source: application-deploy/templates/bookinfo-app.yaml
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: dev-clusters-east
  namespace: bookinfo
spec:
  clusterConditions:
    - type: ManagedClusterConditionAvailable
      status: "True"
  clusterSelector:
    matchLabels:
      env: dev
      region: us-east-1
---
# Source: application-deploy/templates/httpbin-app.yaml
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: dev-clusters-west
  namespace: default
spec:
  clusterConditions:
    - type: ManagedClusterConditionAvailable
      status: "True"
  clusterSelector:
    matchLabels:
      env: dev
      region: us-west-1
---
# Source: application-deploy/templates/bookinfo-app.yaml
apiVersion: apps.open-cluster-management.io/v1
kind: Subscription
metadata:
  name: bookinfo-app
  namespace: bookinfo
  labels:
    app: bookinfo-app
  annotations:
    apps.open-cluster-management.io/github-path: resources/bookinfo
    apps.open-cluster-management.io/git-branch: main
spec:
  channel: bookinfo/bookinfo-app-latest
  placement:
    placementRef:
      kind: PlacementRule
      name: dev-clusters-east
---
# Source: application-deploy/templates/httpbin-app.yaml
apiVersion: apps.open-cluster-management.io/v1
kind: Subscription
metadata:
  name: httpbin-app
  namespace: default
  labels:
    app: httpbin-app
  annotations:
    apps.open-cluster-management.io/github-path: resources/httpbin
    apps.open-cluster-management.io/git-branch: main
spec:
  channel: default/httpbin-app-latest
  placement:
    placementRef:
      kind: PlacementRule
      name: dev-clusters-west
```

Finally, verify the apps were deployed to their respective namespaces on the managed clusters.

Check the `bookinfo` namespace on the East Managed Cluster:
```
k get pods -n bookinfo
```

output
```
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79c697d759-5zbwv       1/1     Running   0          2m51s
productpage-v1-65576bb7bf-f8jph   1/1     Running   0          2m50s
ratings-v1-7d99676f7f-ghvs4       1/1     Running   0          2m51s
reviews-v1-987d495c-qmfzg         1/1     Running   0          2m51s
reviews-v2-6c5bf657cf-kntvz       1/1     Running   0          2m51s
reviews-v3-5f7b9f4f77-mrhqc       1/1     Running   0          2m51s
```

Check the `default` namespace on the West Managed Cluster:
```
k get pods
```

output
```
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-779c54bf49-dcmw5   1/1     Running   0          3m19s
```