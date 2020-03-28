# Open Liberty Application for OpenShift 4.X & IBM Kubernetes with Tekton Pipelines

[Open Liberty image](https://hub.docker.com/r/openliberty/open-liberty-s2i/tags)

[Java Application details](https://github.com/nheidloff/openshift-on-ibm-cloud-workshops/blob/master/2-deploying-to-openshift/documentation/3-java.md#lab-3---understanding-the-java-implementation)

Authors Service APIs - > [http://simple-liberty-app-ci-development.apps.us-west-1.starter.openshift-online.com/openapi/ui/](http://simple-liberty-app-ci-development.apps.us-west-1.starter.openshift-online.com/openapi/ui/)

`.s2i/bin`               folder contains custom s2i scripts for assembling and running the application image.
`.m2/settings.xml`       folder contains custom Maven settings.xml config file.
`liberty-config`         folder contains the Open Liberty server.xml config file.

`openshift-jenkins`      folder contains the Jenkins pipeline implementation and yaml for creating the build config with pipeline strategy.
`openshift-tekton`       folder contains the OpenShift pipeline implementation and yaml for creating the build config with Tekton pipeline strategy.
`kubernetes-tekton`      folder contains the Kubernetes pipeline implementation and yaml for creating the build config with Tekton pipeline strategy.

# Create application image using S2I (source to image) and deploy it 

OC commands:

1.  delete all resources
```
oc delete all -l build=simple-liberty-app
oc delete all -l app=simple-liberty-app
```

2.  create new s2i build config based on openliberty/open-liberty-s2i:19.0.0.12 and image stram
```
oc new-build openliberty/open-liberty-s2i:19.0.0.12 --name=simple-liberty-app --binary=true --strategy=source 
```

3.  create application image from srouce
```
oc start-build bc/simple-liberty-app --from-dir=. --wait=true --follow=true
```

4.  create application based on imagestreamtag : simple-liberty-app:latest
```
oc new-app -i simple-liberty-app:latest
oc expose svc/simple-liberty-app
oc label dc/simple-liberty-app app.kubernetes.io/name=java
```


5.  set readiness and livness probes , and change deploy strategy to Recreate
```
oc set probe dc/simple-liberty-app --readiness --get-url=http://:9080/health --initial-delay-seconds=60
oc set probe dc/simple-liberty-app --liveness --get-url=http://:9080/ --initial-delay-seconds=60
oc patch dc/simple-liberty-app -p '{"spec":{"strategy":{"type":"Recreate"}}}'
```

# OpenShift v4.3 -> CI-CD with OpenShift Pipelines 

Prerequisites : 
- Install OpenShift Pipeline Operator
- Allow pipeline SA to make deploys on other projects :
```
oc create serviceaccount pipeline
oc adm policy add-scc-to-user privileged -z pipeline
oc adm policy add-role-to-user edit -z pipeline
```

OC commands:

1. create Tekton CRDs :
```
oc create -f ci-cd-pipeline/openshift-tekton/resources.yaml
oc create -f ci-cd-pipeline/openshift-tekton/task-build-s2i.yaml
oc create -f ci-cd-pipeline/openshift-tekton/task-test.yaml
oc create -f ci-cd-pipeline/openshift-tekton/task-deploy.yaml
oc create -f ci-cd-pipeline/openshift-tekton/pipeline.yaml
```
2. execute pipeline :
```
tkn t ls
tkn p ls
tkn start liberty-pipeline
```
3. open URI in browser :  
http://<OCP_CLUSTER_HOSTNAME>/health

# OpenShift v4.2 -> CI-CD with Jenkins Pipeline 

Prerequisites : 
- Installed Jenkins template
- Allow jenkins SA to make deploys on other projects :
```
oc policy add-role-to-user edit system:serviceaccount:default:jenkins -n ci-development
```

OC commands:

1. create build configuration resurce in OpenShift :
```
oc create -f  ci-cd-pipeline/openshift-jenkins/liberty-ci-cd-pipeline.yaml 
```

2. create secret for GitLab integration : 
```
oc create secret generic gitlabkey --from-literal=WebHookSecretKey=5f345f345c345
```

3. add webkook to GitLab from Settings->Integration : 

https://<OCP_CLUSTER_HOSTNAME>/apis/build.openshift.io/v1/namespaces/ci-development/buildconfigs/liberty-pipeline-ci-cd/webhooks/5f345f345c345/gitlab

4. start pipeline build or push files into GitLab repo : 
```
oc start-build bc/liberty-pipeline-ci-cd
```

5. get routes for  simple-springboot-app : 
```
oc get routes/simple-liberty-app
```

6. open URI in browser : 
http://<OCP_CLUSTER_HOSTNAME>/health



# IBM Kubernetes 1.16 -> CI-CD with Tekton Pipeline 

kubektl commands:

1. install Tekton pipelines in tekton-pipelines namespace :
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/latest/release.yaml
kubectl get pods --namespace tekton-pipelines
```

2. create new env-dev and env-ci namespaces :
```
kubectl create namespace env-dev
kubectl create namespace env-ci
```

3. create Tekton CRDs :
```
kubectl create -f ci-cd-pipeline/kubernetes-tekton/resources.yaml   -n env-ci
kubectl create -f ci-cd-pipeline/kubernetes-tekton/task-build.yaml  -n env-ci
kubectl create -f ci-cd-pipeline/kubernetes-tekton/task-deploy.yaml -n env-ci
kubectl create -f ci-cd-pipeline/kubernetes-tekton/pipeline.yaml    -n env-ci
```

4. create <API_KEY> for IBM Cloud :
```
ibmcloud iam api-key-create MyKey -d "this is my API key" --file key_file.json
cat key_file.json | grep apikey

kubectl create secret generic ibm-cr-secret  -n env-ci --type="kubernetes.io/basic-auth" --from-literal=username=iamapikey --from-literal=password=<API_KEY>
kubectl annotate secret ibm-cr-secret  -n env-ci tekton.dev/docker-0=us.icr.io
```

5. create service account to allow pipeline run and deploy to env-dev namespace :
```
kubectl apply -f ci-cd-pipeline/kubernetes-tekton/service-account.yaml         -n env-ci
kubectl apply -f ci-cd-pipeline/kubernetes-tekton/service-account-binding.yaml -n env-dev
```

6. execute pipeline via Pipeline Run and watch :
```
kubectl create -f ci-cd-pipeline/kubernetes-tekton/pipeline-run.yaml -n env-ci
kubectl get pipelinerun -n env-ci -w
```

7. check pods and logs :
```
kubectl get pods -n env-dev
kubectl logs liberty-app-76fcdc6759-pjxs7 -f -n env-dev
```

8. open browser with cluster IP and port 32427 :
get Cluster Public IP :
```
kubectl get nodes -o wide
```

http://<CLUSTER_IP>>:32427/health

9. install Tekton Dashboard :
```
kubectl apply -f https://github.com/tektoncd/dashboard/releases/download/v0.5.3/tekton-dashboard-release.yaml
kubectl apply -f ci-cd-pipeline/kubernetes-tekton/tekton-dashboard.yaml -n tekton-pipelines
```

http://<CLUSTER_IP>>:32428/#/pipelineruns



# IBM Kubernetes 1.16 -> Create Tekton WebHooks  for Git


Tekton Trigers, Bindings & EventListeners :

[https://github.com/tektoncd/triggers/blob/master/docs/triggerbindings.md](https://github.com/tektoncd/triggers/blob/master/docs/triggerbindings.md)
[https://github.com/tektoncd/triggers/blob/master/docs/triggertemplates.md](https://github.com/tektoncd/triggers/blob/master/docs/triggertemplates.md)
[https://github.com/tektoncd/triggers/blob/master/docs/eventlisteners.md](https://github.com/tektoncd/triggers/blob/master/docs/eventlisteners.md)

Example :
[https://github.com/tektoncd/triggers/tree/master/examples](https://github.com/tektoncd/triggers/tree/master/examples)


1. install Tekton Triggers :
official release -> [https://github.com/tektoncd/triggers/blob/master/docs/install.md](https://github.com/tektoncd/triggers/blob/master/docs/install.md)
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl get pods --namespace tekton-pipelines
````

2. create SA and Roles and Pipeline Resources :
```
kubectl apply  -f ci-cd-pipeline/kubernetes-tekton/service-account-webhook.yaml -n env-ci
kubectl apply -f ci-cd-pipeline/kubernetes-tekton/service-account-binding-webhook.yaml -n env-ci
```

3. create pipeline's trigger_template, trigger_binding & envent_listener ( in Tekton namespace ! )
```
kubectl apply -f ci-cd-pipeline/kubernetes-tekton/pipeline-run-webhook.yaml -n env-ci 
kubectl get svc  -n env-ci 
kubectl get pods -n env-ci 
kubectl get nodes -o wide
```

4. delete old pipelinerun :
```
kubectl delete pipelinerun liberty-pipeline-run -n env-ci
```

5. forward K8S 8080 port to localhost:8080
```
kubectl port-forward $(kubectl get pod -o=name -l eventlistener=liberty-pipeline-listener -n env-ci) 8080 -n env-ci
```

6. perform a CURL post :
```
curl -X POST \
  http://localhost:8080 \
  -H 'Content-Type: application/json' \
  -H 'X-Hub-Signature: sha1=2da37dcb9404ff17b714ee7a505c384758ddeb7b' \
  -d '{
	"head_commit":
	{
		"id": "master"
	},
	"repository":
	{
		"url": "https://github.com/vladsancira/openliberty-tekton.git"
	}
}'
```

6. watch pipeline run execution and event listener logs:
```
kubectl logs el-liberty-pipeline-listener-7c77666cf7-hzkbf -n env-ci -f
kubectl get pipelinerun -n env-ci -w
tkn pr ls -n env-ci
```


# IBM Kubernetes 1.16 -> Experimental : Tekton Dashboard & WebHook Extension architecture : 

[https://github.com/tektoncd/experimental/blob/master/webhooks-extension/docs/Architecture.md](https://github.com/tektoncd/experimental/blob/master/webhooks-extension/docs/Architecture.md)

1. install Tekton Dashboard :
official release -> [https://github.com/tektoncd/dashboard/releases](https://github.com/tektoncd/dashboard/releases)
```
kubectl apply -f https://github.com/tektoncd/dashboard/releases/download/v0.5.3/tekton-dashboard-release.yaml
kubectl apply -f ci-cd-pipeline/kubernetes-tekton/tekton-dashboard-service.yaml -n tekton-pipelines
```

2. install Tekton WebHook extension :
official release -> [https://github.com/tektoncd/dashboard/releases](https://github.com/tektoncd/dashboard/releases)

```
sed -i '' 's/LOCAL_HOSTNAME/<YOUR_HOSTNAME>/g' ci-cd-pipeline/kubernetes-tekton/tekton-webhooks-extension.yaml
kubectl apply -f ci-cd-pipeline/kubernetes-tekton/tekton-webhooks-extension.yaml
```
3. create webhook from Tekton Dashboard :

![Tekton Dashboard](./ci-cd-pipeline/dashboard.jpg?raw=true "Tekton Dashboard") 

