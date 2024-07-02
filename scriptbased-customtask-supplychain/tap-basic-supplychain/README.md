## procedure to apply custom tasks on custom supply chain for TAP.
testd on TAP 1.8


### design 

#### selector/label matching
selector/label `apps.tanzu.vmware.com/use-custom-basic: "true"` should be matched among follow resources:
- workload -> supply-chain -> clustersourcetemplate -> clusterruntemplate -> task(load env from secrets ) 

selector/label should be mapped to 1:1 among all resources. 

#### Parameter flow
workload -> supply-chain -> clustersourcetemplate -> clusterruntemplate -> task(load env from secrets ) => steps(using param,env)

#### resource scope
cluster resource:
- ClusterSupplyChain
- ClusterSourceTemplate
- ClusterRunTemplate

namespace resource
- Task
- Workload
- RBAC
- secrets

## Procedures

#### (cluster resource) [cluster-run-template.yml](cluster-run-template.yml)

```
apiVersion: carto.run/v1alpha1
kind: ClusterRunTemplate
metadata:
  name: custom-tekton-taskrun
  labels:
    apps.tanzu.vmware.com/use-custom-basic: "true"  # TODO added this as a selector
  ...

```

```
kubectl apply -f cluster-run-template.yml 
k get clusterruntemplate
NAME                        AGE
tekton-custom-taskrun       6s
tekton-source-pipelinerun   8d
```

#### (cluster resource) [cluster-source-template.yml](cluster-source-template.yml)

```
apiVersion: carto.run/v1alpha1
kind: ClusterSourceTemplate
metadata:
  labels:
    apps.tanzu.vmware.com/use-custom-basic: "true" # TODO for custom workload
  annotations:
  #! TODO match name in supply chain
  name: custom-template 
  ...
    apiVersion: carto.run/v1alpha1
    kind: Runnable
    ...
      selector:
        resource:
          apiVersion: tekton.dev/v1beta1
          kind: Task

        #! TODO to match selector
        matchingLabels:
          apps.tanzu.vmware.com/use-custom-basic: "true"
...
```
> add `apps.tanzu.vmware.com/use-custom-basic: "true"` selector.

```
kubectl apply -f cluster-source-template.yml 
k get clustersourcetemplates
NAME                       AGE
custom-template            6s
delivery-source-template   8d
source-scanner-template    8d
source-template            8d
testing-pipeline           8d
```

#### (cluster resource) edit and apply [source-to-url-custom.yml](source-to-url-custom.yml)
create a new `source-to-url-custom` clustersupplychain by duplicate existing `source-to-url` clustersupplychain. please note that it should be fetched the TAP to be used as there are existing TAP dependent params in it

```
k get clustersupplychains source-scan-to-url -o yaml > source-scan-to-url.yml
cp source-scan-to-url.yml source-scan-to-url-custom.yml
```

and update [source-to-url-custom.yml](source-to-url-custom.yml) for parameters according to you environment. 
```
apiVersion: carto.run/v1alpha1
kind: ClusterSupplyChain
metadata:
  name: source-to-url-custom
spec:
...
  resources:
  - name: source-provider
    ...
    templateRef:
      kind: ClusterSourceTemplate
      name: source-template
  ## added
  - name: custom-task
    sources:
    - name: source
      resource: source-provider
    templateRef:
      kind: ClusterConfigTemplate
      name: convention-template

  selector:
    apps.tanzu.vmware.com/use-custom-basic: "true"  # added this as a selector

  selectorMatchExpressions:
  - key: apps.tanzu.vmware.com/workload-type
    operator: In
    values:
    - web
    - server
    - worker
    - custom-server
     ...

```
> rename to unique supplychain name: `source-to-url-custom`
> add `custom-task` task
> add `apps.tanzu.vmware.com/use-custom-basic: "true"` selector.


apply [source-to-url-custom.yml](source-to-url-custom.yml). you have to delete supply chain as it is immutable 
```
kubectl delete -f source-to-url-custom.yml
kubectl apply -f source-to-url-custom.yml
```

```
k get clustersupplychains
NAME                        READY   REASON   AGE
source-test-to-url          True    Ready    6d19h
source-test-to-url-custom   True    Ready    8s
testing-image-to-url        True    Ready    6d19h
```

#### setup developer namespace
```
kubectl create namespace my-space
kubectl label namespaces my-space apps.tanzu.vmware.com/tap-ns=""
```

#### (namespace resource) [task.yml](task.yml)
customize any tasks here such as compile source code, build package, and publish to the any target artifactory such as nexus, jfrog

```
export DEVELOPER_NAMESPACE=my-space

kubectl apply -f task.yml -n $DEVELOPER_NAMESPACE
```
refer to [maven document](https://maven.apache.org/plugins/maven-deploy-plugin/usage.html) for publishing jar artifacts to nexus.

#### (namespace resource) [rbac.yml](../rbac.yml)
you needs to set permission to serviceaccount to list `Task` resources with [rbac.yml](../rbac.yml)
```
export DEVELOPER_NAMESPACE=my-space

kubectl apply -f rbac.yml -n $DEVELOPER_NAMESPACE
```

#### (namespace resource) apply secrets 
artifcatory access credentials. this will be referenced from [task.yml](../task.yml)
```
export DEVELOPER_NAMESPACE=my-space
kubectl apply -f artifactory-credentials.yaml  -n $DEVELOPER_NAMESPACE
```

#### (namespace resource) workload

```
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  name: tanzu-java-web-app
  labels:
    apps.tanzu.vmware.com/workload-type: server ## any workload type defined in supply chain
    app.kubernetes.io/part-of: tanzu-java-web-app
    apps.tanzu.vmware.com/use-custom-basic: "true"  # added this as a scanner task selector
spec:
  source:
    git:
      url: https://github.com/myminseok/tanzu-java-web-app
      ref:
        branch: main
  params:
  - name: artifactory_host
    value: "http://from-workload"
  - name: artifactory_user
    value: "user-from-workload"
```
> add `apps.tanzu.vmware.com/use-custom-basic: "true"` selector.

```
export DEVELOPER_NAMESPACE=my-space
## tanzu apps workload delete tanzu-java-web-app -n ${DEVELOPER_NAMESPACE} 
tanzu apps workload apply -f ./workload-tanzu-java-web-app.yaml --yes   -n ${DEVELOPER_NAMESPACE}
```

#### workload status 
tanzu apps workload get tanzu-java-web-app --namespace my-space
```
Overview
   name:        tanzu-java-web-app
   type:        server
   namespace:   my-space

Source
   type:       git
   url:        https://github.com/myminseok/tanzu-java-web-app
   branch:     main
   revision:   main@sha1:763ba0d6625284113361e7b77216c2c51a2727c2

Supply Chain
   name:   source-to-url-custom

   NAME              READY   HEALTHY   UPDATED   RESOURCE
   source-provider   True    True      12m       gitrepositories.source.toolkit.fluxcd.io/tanzu-java-web-app
   custom-task       True    True      6m16s     runnables.carto.run/tanzu-java-web-app-customtask

Delivery

   Delivery resources not found.

Messages
   No messages found.

Pods
   NAME                                      READY   STATUS      RESTART
S   AGE
   tanzu-java-web-app-c76f9fc8f-wzt4s        1/1     Running     0
    14m
   tanzu-java-web-app-customtask-q4x82-pod   0/2     Completed   0
    6m32s

To see logs: "tanzu apps workload tail tanzu-java-web-app --namespace my
-space --timestamp --since 1h"
```

## workload logs 

tanzu apps workload tail tanzu-java-web-app --namespace my-space
```
...
+ tanzu-java-web-app-customtask-q4x82-pod › step-properties-create
+ tanzu-java-web-app-customtask-q4x82-pod › step-custom-task
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] ##### custom-tekton-task-basic
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create]
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] # /workspace/tmp-workspace/custom-project.properties ---------------------------
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] secret_artifactory_host=https://from-secret
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] artifactory_host=http://from-workload
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] secret_artifactory_user=user_from_secret
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] artifactory_user=user-from-workload
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] secret_artifactory_password=pass_from_secret!
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] artifactory_password=default-pass-from-clustersourcetemplate
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create]
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] # secret_artifactory_ca_crt env detected ---------------------------
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] -----BEGIN CERTIFICATE-----
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] ca+_from_secret
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create] -----END CERTIFICATE-----
tanzu-java-web-app-customtask-q4x82-pod[step-properties-create]
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] ##### custom-tekton-task-basic
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] + wget -qO- http://fluxcd-source-controller.flux-system.svc.cluster.local./gitrepository/my-space/tanzu-java-web-app/763ba0d6625284113361e7b77216c2c51a2727c2.tar.gz
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] + tar xz
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] + cat /workspace/tmp-workspace/custom-project.properties
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] secret_artifactory_host=https://from-secret
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] artifactory_host=http://from-workload
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] secret_artifactory_user=user_from_secret
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] artifactory_user=user-from-workload
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] secret_artifactory_password=pass_from_secret!
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] artifactory_password=default-pass-from-clustersourcetemplate
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] + echo '# importing server CA to truststore ---------------------------'
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] + '[' -f /workspace/tmp-workspace/artifactory_ca.crt ']'
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] + cat /workspace/tmp-workspace/artifactory_ca.crt
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] # importing server CA to truststore ---------------------------
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] -----BEGIN CERTIFICATE-----
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] ca+_from_secret
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task] -----END CERTIFICATE-----
tanzu-java-web-app-customtask-q4x82-pod[step-custom-task]
- tanzu-java-web-app-customtask-q4x82-pod › step-properties-create
- tanzu-java-web-app-customtask-q4x82-pod › step-custom-task
```

## reference
tanzu supply chain(beta): https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.8/tap/supply-chain-about.html
