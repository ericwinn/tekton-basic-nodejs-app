### Simple CICD workflow for a HelloWorld NodeJS in Kubernetes
## Step 1 : Build k8s cluster on OVH

Go to https://www.ovh.com/manager/public-cloud/; Managed Kubernetes Service<br/>
Create a Kubernetes cluster (wait around 5 minuts)<br/>
Then, add two nodes B2-7-FLEX (wait around 10-15 minuts)<br/> 
Then, grap the kubeconfig and paste it on ~/.kube/config<br/>
### Install Tekton
https://github.com/tektoncd/pipeline<br/>
https://github.com/tektoncd/pipeline/blob/master/docs/tutorial.md<br/>
Install tekton cli,tkn
```bash
brew tap tektoncd/tools
brew install tektoncd/tools/tektoncd-cli
``` 
```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```
```bash
kubectl get pods --namespace tekton-pipelines
```
Use your docker.com credentials to push images into your repo.
```bash
kubectl create secret docker-registry docker-creds \
                    --docker-server=index.docker.io \
                    --docker-username=<your-name> \
                    --docker-password=<your-pword> 
```
fork https://github.com/dleurs/tekton-basic-nodejs-app<br/> 
Change helm/values.yaml > tekton.gitSourceResc.gitUrl to your git url forked<br/> 
Change helm/Charts.yaml > version: 0.1.0 and appVersion: 1.0.0 <br/> 

## Step 3 : Installing AlgoCD 
https://argoproj.github.io/argo-cd/getting_started/<br/>
Install algocd on your teminal
```bash
brew tap argoproj/tap
brew install argoproj/tap/argocd
```
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```bash
kubectl get svc argocd-server -n argocd
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
Wait for external IP
```bash
kubectl get svc argocd-server -n argocd
```
Get the password :  (username: admin)
```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```
```bash
argocd login 51.178.XXX.XXX
```
```bash
argocd cluster add # Get the list of k8s contexts
argocd cluster add kubernetes-admin@<cluster name>
```
Open browser, 51.178.XXX.XXX > Advanced parameters > Continue > Username/Password Sign In > New App > Helloworld-nodejs > <br/>
- Application Name : helloworld-nodejs
- Project : default
- Sync policy : Automatic
- Sync option : checked use a schema to validate resource manifest
- Source : (your github forked URL) https://github.com/dleurs/tekton-basic-nodejs-app with GIT
- Path : helm
- Destication, Cluster : kubernetes-admin@<ovh cluster name>
- namespace : default
- in Helm, change image.repository and tekton.gitSourceResc.gitUrl
- Go on top, Create
```bash
argocd app list
argocd app get helloworld-nodejs
argocd app sync helloworld-nodejs
```
On navigator, http://51.178.XX.XXX, => Hello World!
push "AlgoCD working with helm"
```bash
res > main, modify "Hello World!" to "Hello World 2!"
```
```bash
docker build . -t dleurs/helloworld-nodejs:1.0.2
docker push dleurs/helloworld-nodejs:1.0.2
```

helm > chart, appVersion 1.0.2

git push
```bash
argocd app set helloworld-nodejs --sync-policy automated
```
By default (and as a safety mechanism), automated sync will not delete resources when Argo CD detects the resource is no longer defined in Git. To prune the resources, a manual sync can always be performed (with pruning checked). Pruning can also be enabled to happen automatically as part of the automated sync by running
```bash
# argocd app set <APPNAME> --auto-prune 
```



## Other stuff : Using flux => NOT WORKING
https://docs.fluxcd.io/en/latest/tutorials/get-started-helm/
```bash
helm repo add fluxcd https://charts.fluxcd.io
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml
```
```bash
kubectl create namespace flux
```
```bash
helm upgrade -i flux fluxcd/flux \
   --set git.url=git@github.com:dleurs/tekton-basic-nodejs-app \
   --namespace flux
```
```bash
helm upgrade -i helm-operator fluxcd/helm-operator \
   --set git.ssh.secretName=flux-git-deploy \
   --namespace flux
```
```bash
fluxctl identity --k8s-fwd-ns flux
```
Open GitHub, navigate to your fork, go to Setting > Deploy keys, click on Add deploy key, give it a Title, check Allow write access, paste the Flux public key and click Add key.
```bash
```