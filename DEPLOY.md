**Deploy:**

* Create domain records listed in README.md

* Add environment values

```
dashboard_password="Admin1Admin1"
org="myorg"
domain="myorg.com"

* Create Cloudflare API key https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/ for generating Letsencrypt certificates automatically

cloudflare_api_key="ap_key"
```

* Replace all occurrences with your domain

```
find . -type f \( ! -iname "DEPLOY.md" \) -exec sed -i'' -e 's/cryptocat\.eu/'$domain'/g' {} +
find . -type f \( ! -iname "DEPLOY.md" \) -exec sed -i'' -e 's/cryptocat/'$org'/g' {} +
```

## 1. Deploy Cilium

* Build manifest

`kubectl kustomize --enable-helm overlays/cilium/production | kubectl apply --filename -`

* Wait for deployment

`kubectl --namespace cilium rollout status deployment cilium-operator`

## 2. Deploy sealed-secrets

* Download kubeseal CLI

`wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.16.0/kubeseal-linux-amd64 -O /usr/local/bin/kubeseal && chmod +755 /usr/local/bin/kubeseal`

* Create namespace

`kubectl apply -f overlays/sealed-secrets/production/namespace.yaml`

* Create sealed-secrets keys and apply

```
openssl req -x509 -nodes -newkey rsa:4096 -keyout "tls.key" -out "tls.crt" -subj "/CN=sealed-secret/O=sealed-secret"

cat <<EOF | kubectl apply -f -
apiVersion: v1
data:
  tls.crt: $(cat tls.crt | openssl enc -base64 | tr -d '\n')
  tls.key: $(cat tls.key | openssl enc -base64 | tr -d '\n')
kind: Secret
metadata:
  name: sealed-secrets-key
  namespace: sealed-secrets
  labels:
    sealedsecrets.bitnami.com/sealed-secrets-key: active
type: kubernetes.io/tls
EOF

rm tls.key tls.crt
```

* Build manifest and apply

`kubectl kustomize overlays/sealed-secrets/production | kubectl apply --filename -`

* Wait for deployment

`kubectl --namespace sealed-secrets rollout status deployment sealed-secrets-controller`

## 3. Deploy metallb

* Change configmap IP addresses that MetalLB can assign to services

`cat overlays/metallb/production/configmap.yaml`

* Build manifest and apply

`kubectl kustomize overlays/metallb/production | kubectl apply --filename -`

* Wait for deployment

`kubectl --namespace metallb-system rollout status deployment controller`

## 4. Deploy Cert-manager

* Create Cloudflare API key

```
cat <<EOF | kubeseal --controller-namespace=sealed-secrets -o yaml > overlays/cert-manager/production/sealed-secret.yaml
apiVersion: v1
data:
  api-key: $(echo -n "$cloudflare_api_key" | openssl enc -base64)
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
EOF
```

* Set Cloudflare and ACME email for ClusterIssuer

`cat overlays/cert-manager/production/clusterIssuer.yaml`

* Apply CRD manifest

`kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.crds.yaml`

* Build manifest and apply

`kubectl kustomize overlays/cert-manager/production | kubectl apply --filename -`

* Wait for deployment

`kubectl --namespace cert-manager rollout status deployment cert-manager-webhook`

## 5. Deploy Traefik

* Create dashboard secret

```
cat <<EOF | kubeseal --controller-namespace=sealed-secrets -o yaml > overlays/traefik/production/sealed-secret.yaml
apiVersion: v1
data:
  users: $(echo -n $(htpasswd -nb "admin" "$dashboard_password") | openssl enc -base64)
kind: Secret
metadata:
  name: traefik-dashboard-creds
  namespace: traefik
EOF
```

* Apply CRD manifest

`kubectl apply -f https://raw.githubusercontent.com/traefik/traefik-helm-chart/v10.9.1/traefik/crds/ingressroute.yaml -f https://raw.githubusercontent.com/traefik/traefik-helm-chart/v10.9.1/traefik/crds/middlewares.yaml -f https://raw.githubusercontent.com/traefik/traefik-helm-chart/v10.9.1/traefik/crds/tlsstores.yaml`

* Build manifest and apply

`kubectl kustomize --enable-helm overlays/traefik/production | kubectl apply --filename -`

* Wait for deployment

`kubectl --namespace traefik rollout status deployment traefik`

## 6. Deploy linkerd

* Apply namespace, issuer and self-signed certificates

`kubectl apply -f overlays/linkerd/production/namespace.yaml -f overlays/linkerd/production/issuer.yaml -f overlays/linkerd/production/certificate.yaml`

* Create linkerd values

```
cat <<EOF > overlays/linkerd/production/values.yaml
installNamespace: false
identityTrustAnchorsPEM: | 
$(kubectl get secrets -n linkerd linkerd-identity-issuer -o jsonpath="{.data['ca\.crt']}" | base64 -d | sed 's/^/  /')
identity:
  issuer:
    tls:
      crtPEM: |
$(kubectl get secrets -n linkerd linkerd-identity-issuer -o jsonpath="{.data['tls\.crt']}" | base64 -d | sed 's/^/        /')
      keyPEM: |
$(kubectl get secrets -n linkerd linkerd-identity-issuer -o jsonpath="{.data['tls\.key']}" | base64 -d | sed 's/^/        /')
EOF
```
* Build manifest and apply

`kubectl kustomize --enable-helm overlays/linkerd/production | kubectl apply --filename -`

* Create dashboard secret

```
cat <<EOF | kubeseal --controller-namespace=sealed-secrets -o yaml > overlays/linkerd-viz/production/sealed-secret.yaml
apiVersion: v1
data:
  users: $(echo -n $(htpasswd -nb "admin" "$dashboard_password") | openssl enc -base64)
kind: Secret
metadata:
  name: linkerd-viz-dashboard-creds
  namespace: linkerd-viz
EOF
```

## 7. Deploy Rook Ceph

* Apply CRD manifest

`kubectl apply -f https://raw.githubusercontent.com/rook/rook/v1.8.1/deploy/examples/crds.yaml`

* Create dashboard secret

```
cat <<EOF | kubeseal --controller-namespace=sealed-secrets -o yaml > overlays/rook-ceph/production/sealed-secret.yaml
apiVersion: v1
data:
  password: $(echo -n "$dashboard_password" | openssl enc -base64)
kind: Secret
metadata:
  name: rook-ceph-dashboard-password
  namespace: rook-ceph
EOF
```

* Apply namespace and secret

`kubectl apply -f overlays/rook-ceph/production/namespace.yaml -f overlays/rook-ceph/production/sealed-secret.yaml`

* Build manifest and apply

`kubectl kustomize overlays/rook-ceph/production | kubectl apply --filename -`

*  Create test persistent volume claim

`kubectl apply -f overlays/rook-ceph/production/test-pvc.yaml`

* Wait for test PVC to be bound (~15 minutes)

`while [[ $(kubectl get pvc test-pvc -o 'jsonpath={..status.phase}') != "Bound" ]]; do echo "waiting for PVC status" && sleep 30; done`

## 8. Deploy Gitlab

* Create dashboard secret

```
cat <<EOF | kubeseal --controller-namespace=sealed-secrets -o yaml > overlays/gitlab/production/sealed-secret.yaml
apiVersion: v1
data:
  password: $(echo -n "$dashboard_password" | openssl enc -base64)
kind: Secret
metadata:
  name: gitlab-root-password
  namespace: gitlab
EOF
```

* Apply namespace and dashboard secret

`kubectl apply -f overlays/gitlab/production/namespace.yaml -f overlays/gitlab/production/sealed-secret.yaml`

* Apply shared-secrets job

`kubectl apply -f overlays/gitlab/production/sharedsecrets.yaml`

* Wait for shared-secrets job completion

```
kubectl wait --for=condition=complete -n gitlab job/gitlab-shared-secrets
kubectl wait --for=condition=complete -n gitlab job/gitlab-shared-secrets-selfsign
```

* Build manifest and apply

`kubectl kustomize --enable-helm overlays/gitlab/production | kubectl apply -f -`

* Wait for deployment (~10 minutes)

`kubectl rollout status deployment -n gitlab gitlab-webservice-default`

**Gitlab Items**

```
* Get internal ip
gitlab_ip=$(kubectl get svc -n gitlab gitlab-webservice-default -o=jsonpath='{.spec.clusterIP}')

* Generate Token (~1 minutes)
kubectl -n gitlab -c toolbox exec $(kubectl get pod -n gitlab | awk '{print $1}' | grep toolbox) -- gitlab-rails runner "token = User.find_by_username('root').personal_access_tokens.create(scopes: [:api], name: 'Automation token'); token.set_token('$dashboard_password'); token.save!"

* Create Group
curl -H "PRIVATE-TOKEN: $dashboard_password" http://$gitlab_ip:8181/api/v4/groups/ -X POST -H "Content-Type: application/json" -d '{"name": "'$org'", "path": "'$org'", "description": "Org Group", "visibility": "private" }'

* Create Project - "core"
curl -H "PRIVATE-TOKEN: $dashboard_password" "http://$gitlab_ip:8181/api/v4/projects?name=core&visibility=private&namespace_id=3" -X POST
```

##  9. Deploy ArgoCD

* Create ArgoCD configuration

```
cat <<EOF > overlays/argo-cd/production/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  kustomize.buildOptions: --load-restrictor LoadRestrictionsNone --enable-helm
  url: https://argo-cd.$domain
  dex.config: |
    connectors:
      - type: gitlab
        id: gitlab
        name: GitLab
        config:
          baseURL: https://gitlab.$domain
          clientSecret: $(curl -s -H "PRIVATE-TOKEN: $dashboard_password" http://$gitlab_ip:8181/api/v4/applications -X POST -H "Content-Type: application/json" -d '{"name": "ArgoCD", "redirect_uri": "https://argo-cd.'$domain'/api/dex/callback", "scopes": "api read_user openid" }' | jq -r '.secret')
          clientID: $(curl -s -H "PRIVATE-TOKEN: $dashboard_password" http://$gitlab_ip:8181/api/v4/applications | jq  -r 'map(select(.application_name == "ArgoCD")) | last | .application_id')
          redirectURI: $(curl -s -H "PRIVATE-TOKEN: $dashboard_password" http://$gitlab_ip:8181/api/v4/applications | jq  -r 'map(select(.application_name == "ArgoCD")) | last | .callback_url')
          groups:
          - $org
        useLoginAsID: false
    staticClients:
      - id: argo-workflows-sso
        name: Argo Workflow
        redirectURIs:
          - https://argo-workflows.$domain/oauth2/callback
        secretEnv: ARGO_WORKFLOWS_SSO_CLIENT_SECRET
EOF
```

* Create dashboard secret

```
cat <<EOF | kubeseal --controller-namespace=sealed-secrets -o yaml > overlays/argo-cd/production/sealed-secret.yaml
apiVersion: v1
data:
  admin.password: $(echo -n $(htpasswd -bnBC 10 "" "$dashboard_password") | tr -d ':\n' | openssl enc -base64 | tr -d '\n')
  server.secretkey: "$(date +%s | sha256sum | base64 | head -c 32 ; echo | openssl enc -base64)"
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
EOF
```

* Create repository secrets

```
cat <<EOF | kubeseal --controller-namespace=sealed-secrets -o yaml > repos-core.yaml
apiVersion: v1
data:
  name: $(echo -n core | openssl enc -base64)
  url: $(echo -n "https://gitlab.$domain/$org/core.git" | openssl enc -base64)
  password: $(echo -n "$dashboard_password" | openssl enc -base64)
  username: $(echo -n root | openssl enc -base64)
  insecure: $(echo -n true | openssl enc -base64)
kind: Secret
metadata:
  name: gitlab-repo-core
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
EOF
```

* Create workflows secrets

```
cat <<EOF | kubeseal --controller-namespace=sealed-secrets -o yaml > overlays/argo-cd/production/workflows-sealed-secret.yaml
apiVersion: v1
data:
  client-id: $(echo -n "argo-workflows-sso" | openssl enc -base64)
  client-secret: $(uuidgen | openssl enc -base64)
kind: Secret
metadata:
  name: argo-workflows-sso
  namespace: argocd
EOF
```

* Build manifest and apply

`kubectl kustomize overlays/argo-cd/production | kubectl apply --filename -`

* Wait for deployment

`kubectl --namespace argocd rollout status deployment argocd-server`

* Apply ArgoCD objects

```
kubectl apply -f repos-core.yaml
kubectl apply --filename apps.yaml
kubectl apply --filename projects.yaml
```

10.  Commit to Gitlab

```
git init
git add .
git config user.name "$org"
git config user.email "$org@$domain"
git config --global http.sslVerify false
git commit -m "upstream push"
git remote add origin "https://root:$dashboard_password@gitlab.$domain/$org/core"
git push -f origin master
```
