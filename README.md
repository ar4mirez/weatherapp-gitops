== Tekton with ArgoCD

Blueprint for a Tekton + ArgoCD application setup.

=== Installation

Requires a Kubernetes cluster with Istio installation.
You can for example create one on the https://ibm.biz/cloud-reg-istio-ws[IBM Cloud^].

Install Tekton:

----
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
----

Create secrets for SSH GitHub and Docker registry access:

This is one way to create a SSH key, that will be added to GitHub.
There are in fact many ways for Tekton to get access to GitHub.

----
ssh-keygen -t rsa -b 4096 -C "tekton@tekton.dev"
# save as tekton / tekton.pub
# add tekton.pub contents to GitHub

# create secret YAML from contents
cat tekton | base64 -w 0
cat > tekton-git-ssh-secret.yaml << EOM
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh-key
  namespace: tekton-pipelines
  annotations:
    tekton.dev/git-0: github.com
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64 data>
---
EOM

kubectl apply -f tekton-git-ssh-secret.yaml

# create your Docker registry secret, for example:
cat ~/.docker/config.json | base64 -w 0
cat > regsecret.yaml << EOM
kind: Secret
apiVersion: v1
metadata:
  name: regsecret
data:
  .dockercfg: <base 64 data>
type: kubernetes.io/dockercfg
EOM

kubectl apply -n tekton-pipelines -f regsecret.yaml
----

Setup the Tekton serviceaccount:

----
kubectl apply -f tekton/
----

Install the Tekton dashboard:

----
kubectl apply -f https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml
----

Install ArgoCD:

----
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f argocd/
kubectl apply -n systemtest -f regsecret.yaml
kubectl apply -n production -f regsecret.yaml
----

Login in ArgoCD, find out the admin password and create a token:

----
# pod name is the admin password
kubectl get pods -n argocd | grep argocd-server

# forward ports to access in browser
kubectl -n argocd port-forward svc/argocd-server 8081:80
kubectl -n tekton-pipelines port-forward svc/tekton-dashboard 9097:9097
----

Create ArgoCD access with Tekton:

Under https://localhost:8081/settings/accounts/tekton generate a token.

----
kubectl create secret -n tekton-pipelines generic argocd-env-secret '--from-literal=ARGOCD_AUTH_TOKEN=<token>'
----

Now, adapt all ocurrences of your application and GitOps config repository, and your application Docker image.

Then you can execute the pipeline, manually:

----
./pipelinerun/trigger-pipeline.sh
----

=== Tekton Triggers

You can setup Tekton Triggers that start the build on a push to the repository `main` branch.

Install Tekton Triggers:

----
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply -f pipelinetriggers/
----

Create a triggers secret for GitHub:

----
cat > github-trigger-secret.yaml << EOM
apiVersion: v1
kind: Secret
metadata:
  name: github-trigger-secret
  namespace: tekton-pipelines
type: Opaque
stringData:
  secretToken: "123"
---
EOM

kubectl apply -f github-trigger-secret.yaml
----

Test the triggers setup manually:

----
# HMAC is generated from payload and the GitHub triggers secret
curl -i \
  -H 'X-GitHub-Event: push' \
  -H 'X-Hub-Signature: sha1=<HMAC>' \
  -H 'Content-Type: application/json' \
  -d '{"ref":"refs/heads/main","head_commit":{"id":"123abc..."}}' \
  http://tekton-triggers.example.com
----

After you've setup a GitHub WebHook for push events, you can test the pipeline via pushing to you application repository.