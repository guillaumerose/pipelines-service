

# KCP in Openshift!
## and we call it _ckcp_ : containerized-kcp

###
### Description

This script essentially does this :  

Short Version:
1. Run kcp in a container in an Openshift cluster.
2. Add the current cluster as a physical cluster when running KCP in k8s.
3. Install pipelines and/or triggers (optional)

Long Version:
1. Create ns, sa, and add appropriate scc.
2. Create deployment and service resources.
3. Copy kube config from inside the pod to local system.
4. Update external IP Address of kcp-service in the just copied admin.kubeconfig file.
5. Copy a physical cluster's kubeconfig inside a pod.
6. Add a physical cluster to kcp running inside the pod.  
   ***(optional)***
7. Apply patches to pipelines repo and run the controller.
8. Run some example TaskRuns and PipelineRuns.
9. Apply patches to triggers repo and run the controller, interceptor and eventlistener.

### Pre-requisites
Before you execute the script, 

1. You need to have a kubernetes/openshift cluster.
2. You need to build the docker image for ckcp and upload it to *quay.io* and make it *Public*.  
Note: KO_DOCKER_REPO has to point to your own *quay.io* repo.
```
$ echo $KO_DOCKER_REPO
quay.io/bnr
```

```
$ ./build.sh
```

Yup that's it! Just run the script, and you're ready with a pod running kcp inside it!  

You can run the run.sh script with or without parameters as specified below:  
Note: Triggers requires pipelines to be running and thus running ckcp with triggers alone is not supported.

```
$ ./run.sh
    #installs kcp in a pod running in a k8s cluster a.k.a ckcp

$ ./run.sh pipelines
    #installs ckcp + pipelines

$ ./run.sh pipelines triggers
    #installs ckcp + pipelines + triggers 
```

test.sh script runs certain examples from tektoncd repo for pipelines and triggers. You can run the below script only after run.sh is run and the required resources are up and running. 

```
$ ./test.sh pipelines
    #Runs TaskRun and PipelineRun which sets and uses some env variables respectively.
    #https://github.com/tektoncd/pipeline/blob/main/examples/v1beta1/taskruns/custom-env.yaml
    #https://github.com/tektoncd/pipeline/blob/main/examples/v1beta1/pipelineruns/using_context_variables.yaml

$ ./run.sh triggers
    #Simulates a webhook for a Github PR which triggers a TaskRun
    #https://github.com/tektoncd/triggers/tree/main/examples/v1beta1/github

$ ./run.sh pipelines triggers
    #Runs both tests
```

Once the script is done executing, notice that _deployment.apps, pods_ and _service_ (we specified only these 3 when we started kcp) are synced after we registered our physical cluster with kcp.

<pre>
 $ KUBECONFIG=kubeconfig/admin.kubeconfig kubectl api-resources
NAME                          SHORTNAMES   APIVERSION                             NAMESPACED   KIND
configmaps                    cm           v1                                     true         ConfigMap
events                        ev           v1                                     true         Event
limitranges                   limits       v1                                     true         LimitRange
namespaces                    ns           v1                                     false        Namespace
<b>pods                          po           v1                                     true         Pod</b>
resourcequotas                quota        v1                                     true         ResourceQuota
secrets                                    v1                                     true         Secret
serviceaccounts               sa           v1                                     true         ServiceAccount
<b>services                      svc          v1                                     true         Service</b>
customresourcedefinitions     crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
apiresourceimports                         apiresource.kcp.dev/v1alpha1           false        APIResourceImport
negotiatedapiresources                     apiresource.kcp.dev/v1alpha1           false        NegotiatedAPIResource
<b>deployments                   deploy       apps/v1                                true         Deployment</b>
tokenreviews                               authentication.k8s.io/v1               false        TokenReview
localsubjectaccessreviews                  authorization.k8s.io/v1                true         LocalSubjectAccessReview
selfsubjectaccessreviews                   authorization.k8s.io/v1                false        SelfSubjectAccessReview
selfsubjectrulesreviews                    authorization.k8s.io/v1                false        SelfSubjectRulesReview
subjectaccessreviews                       authorization.k8s.io/v1                false        SubjectAccessReview
certificatesigningrequests    csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
clusters                                   cluster.example.dev/v1alpha1           false        Cluster
leases                                     coordination.k8s.io/v1                 true         Lease
events                        ev           events.k8s.io/v1                       true         Event
flowschemas                                flowcontrol.apiserver.k8s.io/v1beta1   false        FlowSchema
prioritylevelconfigurations                flowcontrol.apiserver.k8s.io/v1beta1   false        PriorityLevelConfiguration
pods                                       metrics.k8s.io/v1beta1                 true         PodMetrics
clusterrolebindings                        rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                               rbac.authorization.k8s.io/v1           false        ClusterRole
rolebindings                               rbac.authorization.k8s.io/v1           true         RoleBinding
roles                                      rbac.authorization.k8s.io/v1           true         Role
workspaces                                 tenancy.kcp.dev/v1alpha1               false        Workspace
workspaceshards                            tenancy.kcp.dev/v1alpha1               false        WorkspaceShard
</pre>

### Get access on an already setup cluster

Configure kubectl to point to your physical cluster and run:

```
$ kubectl get secret ckcp-kubeconfig -n ckcp  -o jsonpath="{.data['admin\.kubeconfig']}" > kubeconfig
```
