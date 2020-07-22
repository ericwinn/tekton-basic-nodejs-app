### Simple HelloWorld NodeJS to run in Kubernetes
## Step 0 : Build k8s cluster on OVH
You can also test the project locally
```bash
npm install
npm run
```
## Step 1 : Just with helm
```bash
docker build . -t dleurs/helloworld-nodejs:1.0.1
docker push dleurs/helloworld-nodejs:1.0.1
```
```bash
helm install helloworld-nodejs ./helm # check helm/values.yaml (image.repository) and helm/Chart.yaml (appVersion)
# corresponding to docker image pushed
```
```bash
helm ls
helm delete helloworld-nodejs
```
## Step 2 : Using flux NOT WORKING
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

## Step 2 : Using algocd
https://argoproj.github.io/argo-cd/getting_started/
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
```bash
brew tap argoproj/tap
brew install argoproj/tap/argocd
```
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
Get the password :  (username: admin)
```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```
```bash
k get svc -n argocd
argocd login 51.178.XXX.XXX
```
```bash
argocd cluster add
argocd cluster add kubernetes-admin@ovh-k8s
```
Open browser, 51.178.XXX.XXX > New App > Helloworld-nodejs > <br/>
https://github.com/dleurs/tekton-basic-nodejs-app on GIT
path : helm

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
```