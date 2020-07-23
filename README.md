# Simple CICD workflow for a HelloWorld NodeJS in Kubernetes
GitOps / Kubernetes / Docker / ArgoCD / Helm / NodeJS / TypeScript
## Step 1 : Build k8s cluster on OVH

Go to https://www.ovh.com/manager/public-cloud/; Managed Kubernetes Service<br/>
Create a Kubernetes cluster (wait around 5 minuts)<br/>
Then, add two nodes B2-7-FLEX (wait around 10-15 minuts)<br/> 
Then, grap the kubeconfig and paste it on ~/.kube/config<br/>
## Step 2 : Install Tekton
https://github.com/tektoncd/pipeline<br/>
https://github.com/tektoncd/pipeline/blob/master/docs/tutorial.md<br/>
Install tekton cli, tkn :
```bash
brew tap tektoncd/tools
brew install tektoncd/tools/tektoncd-cli
``` 
Install tekton on cluster : 
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
Change helm/values.yaml :
- tekton.gitSourceResc.gitUrl to your git url forked
- image.repository to docker repo
- image.tag to 1.0.0
- knService.use : false
Change helm/Charts.yaml > version: 0.1.0 <br/> 

If you are using a private git repo, please look at :<br/>
https://github.com/dleurs/tekton-basic-nodejs-app-private-repo


Get helm cli : https://helm.sh/docs/intro/install/
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
# Test the repo
cd forked_github_repo
helm template ./helm
```
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
In your github folked
```bash
git add -A; git commit -m "commit msg : initial commit, Hello World"; git push origin master;
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
- in Helm, check 
- Go on top, Create
```bash
argocd app list
argocd app get helloworld-nodejs
argocd app sync helloworld-nodejs

tkn taskrun list
tkn taskrun describe build-task-run-1-0-0
tkn taskrun logs build-task-run-1-0-0
```
```bash
kubectl get svc argocd-server -n argocd
```
On navigator, algocd interface, you will be able to see Deployment, service and tekton taskrun<br/>
```bash
kubectl get svc helloworld-nodejs-svc 
```
```bash
curl 51.178.XXX.XXX # Hello World !
```

## Step 4 : Test CICD

In res > main, modify "Hello World!" to "Hello World 2!"<br/>

In helm > Values, change image.tag from 1.0.0 to 1.0.1 <br/>
In helm > Chart, version: from 0.1.0 to 0.1.1 <br/>
```bash
git add -A; git commit -m "commit msg : Hello World 1 now"; git push origin master
```
```bash
tkn taskrun list
tkn taskrun describe build-task-run-1-0-1
tkn taskrun logs build-task-run-1-0-1
kubectl get pod
```
```bash
curl 51.178.XXX.XXX # Hello World 2!
```

## Go Beyond - Step 5 - Install Istio and Knative Serving v0.16

https://knative.dev/docs/install/any-kubernetes-cluster/

Install the Custom Resource Definitions :
```bash
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.16.0/serving-crds.yaml
```
Install the core components of Serving :
```bash
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.16.0/serving-core.yaml
```
Install istioctl on your laptop terminal
```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.6.5
export PATH=$PWD/bin:$PATH
```
Install Istio 1.6.5 for Knative with sidecar injection
```bash
cat << EOF > ./istio-minimal-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      proxy:
        autoInject: enabled
      useMCP: false
      # The third-party-jwt is not enabled on all k8s.
      # See: https://istio.io/docs/ops/best-practices/security/#configure-third-party-service-account-tokens
      jwtPolicy: first-party-jwt

  addonComponents:
    pilot:
      enabled: true
    prometheus:
      enabled: false

  components:
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
      - name: cluster-local-gateway
        enabled: true
        label:
          istio: cluster-local-gateway
          app: cluster-local-gateway
        k8s:
          service:
            type: ClusterIP
            ports:
            - port: 15020
              name: status-port
            - port: 80
              name: http2
            - port: 443
              name: https
EOF
```
```bash
istioctl manifest apply -f istio-minimal-operator.yaml # around 2-3 minuts
/bin/rm -rf istio-minimal-operator.yaml
```
```bash
kubectl label namespace knative-serving istio-injection=enabled
```
```bash
cat <<EOF | kubectl apply -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "knative-serving"
spec:
  mtls:
    mode: PERMISSIVE
EOF
```
Knative Istio controller
```bash
kubectl apply --filename https://github.com/knative/net-istio/releases/download/v0.16.0/release.yaml
```
### Checking the install
```bash
kubectl get pods --namespace istio-system
kubectl get pods --namespace knative-serving
```

### Test with a hello world
```bash
cat <<EOF | kubectl apply -f -
apiVersion: serving.knative.dev/v1 # Current version of Knative
kind: Service
metadata:
  name: helloworld-go # The name of the app
  namespace: default # The namespace the app will use
  #labels:
  #  serving.knative.dev/visibility: cluster-local
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1" # No scale to zero
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go # Reference to the image of the app
          env:
            - name: TARGET # The environment variable printed out by the sample app
              value: "Go Sample v1"
EOF
```

## Go Beyond - Step 6 - HelloWorld NodeJS with Knative Serving

Helm > values.yaml > knService.use : true
```bash
git add -A; git commit -m "Use Knative and Istio"; git push origin master;
```

```bash
kubectl get ksvc
kubectl --namespace istio-system get service istio-ingressgateway # Take External IP 51.178.XXX.XXX
curl -H "Host: helloworld-nodejs.default.example.com" http://51.178.XXX.XXX # Hello World
```

## Go Beyond - Step 7 - Setup DNS
https://www.ovh.com/manager/web/<br/>
You will need a domain for the following, you can buy one on OVH, a .fr cost 5 euros.<br/>
Here we will use mydomain.fr as an example<br/>
First, get external IP of Istio
```bash
kubectl --namespace istio-system get service istio-ingressgateway # Take External IP 51.178.XXX.XXX
```
Go to OVH Web > Domains > mydomain.fr > DNS zone > Add an entry > A
- Sub-domain : *.default  .mydomain.fr
- Target : 51.178.XXX.XXX
*.default IN A 51.178.XXX.XXX

kubectl edit cm config-domain --namespace knative-serving
```
```bash
apiVersion: v1
data:
  mydomain.fr: ""
kind: ConfigMap
[...]
```

```bash
kubectl get ksvc
curl http://helloworld-nodejs.default.mydomain.fr # <h1>Hello World!</h1>
```
Same for argocd server
```bash
kubectl get svc argocd-server -n argocd # 51.178.XXX.XXX
```
Go to OVH Web > Domains > mydomain.fr > DNS zone > Add an entry > A

Sub-domain : algocd .mydomain.fr
Target : 51.178.XXX.XXX

```bash
On browser : http://argocd.mydomain.fr/
```

## To go beyond -  Step 6 : Setup HTTPS, Cert Manager and Let's Encrypt

### Install cert-manager
https://cert-manager.io/docs/installation/kubernetes/#installing-with-helm
```bash
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
```
```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager-legacy.crds.yaml

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.1
```
```bash
kubectl get pods --namespace cert-manager
```
```bash
```
