## procedure to apply custom tasks on custom supply chain for TAP.
testd on TAP 1.8


#### selector/label matching
selector/label `apps.tanzu.vmware.com/use-custom: "true"` should be matched among follow resources:
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
kubectl apply -f cluster-run-template.yml 
k get clusterruntemplate
NAME                        AGE
tekton-custom-taskrun       6s
tekton-source-pipelinerun   8d
```

#### (cluster resource) [cluster-source-template.yml](cluster-source-template.yml)
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

#### (cluster resource) edit and apply [source-test-to-url-custom.yml](source-test-to-url-custom.yml)
create a new `source-test-to-url-custom` clustersupplychain by duplicate existing `source-test-to-url` clustersupplychain. please note that it should be fetched the TAP to be used as there are existing TAP dependent params in it

```
k get clustersupplychains source-test-scan-to-url -o yaml > source-test-scan-to-url.yml
cp source-test-scan-to-url.yml source-test-scan-to-url-custom.yml
```

and update [source-test-to-url-custom.yml](source-test-to-url-custom.yml) for parameters according to you environment. 
```
apiVersion: carto.run/v1alpha1
kind: ClusterSupplyChain
metadata:
  name: source-test-to-url-custom
spec:
...
  resources:
  - name: source-provider
    ...
    templateRef:
      kind: ClusterSourceTemplate
      name: source-template

  - name: source-tester
    sources:
    - name: source
      resource: source-provider
    templateRef:
      kind: ClusterSourceTemplate
      name: testing-pipeline

  ## added
  - name: custom-task
    sources:
    - name: source
      resource: source-tester
    templateRef:
      kind: ClusterSourceTemplate
      name: custom-template

  selector:
    apps.tanzu.vmware.com/has-tests: "true"
    apps.tanzu.vmware.com/use-custom: "true"  # added this as a selector

  selectorMatchExpressions:
  - key: apps.tanzu.vmware.com/workload-type
    operator: In
    values:
    - web
    - server
    - worker
    - custom-server

```
> rename to unique supplychain name: `source-test-to-url-custom`
> add `custom-task` task
> add `apps.tanzu.vmware.com/use-custom: "true"` selector.


apply [source-test-to-url-custom.yml](source-test-to-url-custom.yml). you have to delete supply chain as it is immutable  
```
kubectl delete -f source-test-to-url-custom.yml
kubectl apply -f source-test-to-url-custom.yml
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
kubectl apply -f tekton-pipeline-testing-pipeline.yml -n $DEVELOPER_NAMESPACE
```
#### (namespace resource) [tekton-pipeline-testing-pipeline.yml](./tekton-pipeline-testing-pipeline.yml)
add tekton pipline for testing
```
export DEVELOPER_NAMESPACE=my-space
kubectl apply -f tekton-pipeline-testing-pipeline.yml -n $DEVELOPER_NAMESPACE
```

#### (namespace resource) apply secrets 
artifcatory access credentials. this will be referenced from [task.yml](../task.yml)
```
export DEVELOPER_NAMESPACE=my-space
kubectl apply -f artifactory-credentials.yaml  -n $DEVELOPER_NAMESPACE
```
#### (namespace resource) workload
```
export DEVELOPER_NAMESPACE=my-space
# tanzu apps workload delete tanzu-java-web-app -n ${DEVELOPER_NAMESPACE} 
tanzu apps workload apply -f ./workload-tanzu-java-web-app.yaml --yes   -n ${DEVELOPER_NAMESPACE}
```

#### workload status 
tanzu apps workload get tanzu-java-web-app --namespace my-space
```
Overview
   name:        tanzu-java-web-app
   type:        web
   namespace:   my-space

Source
   type:       git
   url:        https://github.com/myminseok/tanzu-java-web-app
   branch:     main
   revision:   main@sha1:763ba0d6625284113361e7b77216c2c51a2727c2

Supply Chain
   name:   source-test-to-url-custom

   NAME              READY   HEALTHY   UPDATED   RESOURCE
   source-provider   True    True      45m       gitrepositories.source.toolkit.fluxcd.io/tanzu-java-web-app
   source-tester     True    True      45m       runnables.carto.run/tanzu-java-web-app
   custom-task       True    True      6m21s     runnables.carto.run/tanzu-java-web-app-customtask

Delivery

   Delivery resources not found.

Messages
   No messages found.

Pods
   NAME                                      READY   STATUS      RESTARTS   AGE
   tanzu-java-web-app-48bvd-test-pod         0/1     Completed   0          45m
   tanzu-java-web-app-customtask-4nlqj-pod   0/2     Completed   0          6m38s

To see logs: "tanzu apps workload tail tanzu-java-web-app --namespace my-space --timestamp --since 1h"
```

## workload logs 

tanzu apps workload tail tanzu-java-web-app --namespace my-space
```
...
+ tanzu-java-web-app-customtask-4nlqj-pod › step-custom-task
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create]
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] # /workspace/tmp-workspace/custom-project.properties ---------------------------
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] secret_artifactory_host=https://from-secret
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] artifactory_host=http://from-workload
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] secret_artifactory_user=user_from_secret
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] artifactory_user=user-from-workload
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] secret_artifactory_password=pass_from_secret!
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] artifactory_password=default-pass-from-clustersourcetemplate
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create]
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] # secret_artifactory_ca_crt env detected ---------------------------
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] -----BEGIN CERTIFICATE-----
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] ca+_from_secret
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create] -----END CERTIFICATE-----
tanzu-java-web-app-customtask-4nlqj-pod[step-properties-create]
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] + wget -qO- http://fluxcd-source-controller.flux-system.svc.cluster.local./gitrepository/my-space/tanzu-java-web-app/763ba0d6625284113361e7b77216c2c51a2727c2.tar.gz
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] + tar xz
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] + cat /workspace/tmp-workspace/custom-project.properties
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] secret_artifactory_host=https://from-secret
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] artifactory_host=http://from-workload
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] secret_artifactory_user=user_from_secret
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] artifactory_user=user-from-workload
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] secret_artifactory_password=pass_from_secret!
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] artifactory_password=default-pass-from-clustersourcetemplate
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] + echo '# importing server CA to truststore ---------------------------'
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] + '[' -f /workspace/tmp-workspace/artifactory_ca.crt ']'
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] + cat /workspace/tmp-workspace/artifactory_ca.crt
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] # importing server CA to truststore ---------------------------
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] -----BEGIN CERTIFICATE-----
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] ca+_from_secret
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task] -----END CERTIFICATE-----
tanzu-java-web-app-customtask-4nlqj-pod[step-custom-task]
```

## reference
tanzu supply chain(beta): https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.8/tap/supply-chain-about.html
