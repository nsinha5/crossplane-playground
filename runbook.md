minikube start

helm repo add crossplane-stable \
    https://charts.crossplane.io/stable
helm repo update

helm upgrade --install \
    crossplane crossplane-stable/crossplane \
    --namespace crossplane-system \
    --create-namespace \
    --wait

#In lens notice image pull off in crossplane and crossplane-rbac-manager pods for image crossplane/crossplane:v1.12.2 & the helm command will also take long time and then throw an error.
#Run below command to fix it
minikube image load crossplane/crossplane:v1.12.2

# In few seconds the pods will be running 
kubectl apply \
    --filename crossplane-config/provider-kubernetes-incluster.yaml

# Notice that Provider will be created but not shown as INSTALLED AND HEALTHY in lens
#NAME                             INSTALLED   HEALTHY   PACKAGE                                                         AGE
#crossplane-provider-kubernetes                         xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.7.0   81s


# Notice below error in the output of kubectl describe
# kubectl describe provider crossplane-provider-kubernetes
# Type     Reason         Age                  From                                 Message
#  ----     ------         ----                 ----                                 -------
#  Warning  UnpackPackage  13s (x8 over 2m20s)  packages/provider.pkg.crossplane.io  cannot unpack package: failed to fetch package digest from remote: failed to fetch package descriptor with a GET request after a previous HEAD request failure: Get "https://xpkg.upbound.io/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority: Get "https://xpkg.upbound.io/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority

# Apply BLK cert as config map to namespace crossplane-system
kubectl apply -f ~/Downloads/cert-cm.yaml -n crossplane-system

# Teach crossplane to use it
helm upgrade crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace \
--set provider.packages={crossplane/provider-kubernetes:main} \
--set registryCaBundleConfig.name="ca-bundle" --set registryCaBundleConfig.key="ca-bundle"


# Notice Provider still have the below issue
#  Warning  UnpackPackage  41s (x19 over 13m)  packages/provider.pkg.crossplane.io  cannot unpack package: failed to fetch package digest from remote: failed to fetch package descriptor with a GET request after a previous HEAD request failure: Get "https://xpkg.upbound.io/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority: Get "https://xpkg.upbound.io/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority
#  Warning  UnpackPackage  28s                 packages/provider.pkg.crossplane.io  cannot unpack package: failed to fetch package digest from remote: failed to fetch package descriptor with a GET request after a previous HEAD request failure: Get "https://index.docker.io/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority: Get "https://index.docker.io/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority

# Add below proxies to your crossplane Deployment (I used lens to edit crossplane and crossplane-provider-kubernetes deployments)
#   HTTP_PROXY: http://application-proxy.blackrock.com:9443
#    HTTPS_PROXY: http://application-proxy.blackrock.com:9443
#    NO_PROXY: privatelink.westeurope.azmk8s.io,privatelink.northeurope.azmk8s.io,privatelink.westus2.azmk8s.io,privatelink.eastus2.azmk8s.io,29.12.192.194,.blackrock.com,.na.blkint.com,localhost,127.0.0.1,argocd-repo-server,argocd-application-controller,argocd-metrics,argocd-server,argocd-server-metrics,argocd-redis,argocd-dex-server,10.0.0.0/8
#    http_proxy: http://application-proxy.blackrock.com:9443
#    https_proxy: http://application-proxy.blackrock.com:9443
#    no_proxy: privatelink.westeurope.azmk8s.io,privatelink.northeurope.azmk8s.io,privatelink.westus2.azmk8s.io,privatelink.eastus2.azmk8s.io,29.12.192.194,.blackrock.com,.na.blkint.com,localhost,127.0.0.1,argocd-repo-server,argocd-application-controller,argocd-metrics,argocd-server,argocd-server-metrics,argocd-redis,argocd-dex-server,10.0.0.0/8

# Then noticed below error in the crossplane-provider pod
#Failed to pull image "crossplane/provider-kubernetes-controller:v0.5.0-rc.13.g7d48dad": rpc error: code = Unknown desc = Error response from daemon: Head "https://registry-1.docker.io/v2/crossplane/provider-kubernetes-controller/manifests/v0.5.0-rc.13.g7d48dad": Get "https://auth.docker.io/token?scope=repository%3Acrossplane%2Fprovider-kubernetes-controller%3Apull&service=registry.docker.io": x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "BLACKROCK ROOT CA")Source
#Run below command to fix it

#Tips - docker pull might bring a different version, hence you might need to adjust the image name
docker image pull crossplane/provider-kubernetes-controller:v0.5.0-rc.13.g7d48dad
minikube image load crossplane/provider-kubernetes-controller:v0.5.0-rc.13.g7d48da

#YAY , Happy Provider now
#nasinha@ATLLA143MD6T devops-toolkit-crossplane % kubectl get providers -n crossplane-system                                    
#NAME                             INSTALLED   HEALTHY   PACKAGE                               AGE
#crossplane-provider-kubernetes   True        True      crossplane/provider-kubernetes:main   62m

kubectl apply \
    --filename crossplane-config/config-app.yaml

kubectl create ns a-team

kubectl --namespace a-team apply \
    --filename examples/app-frontend.yaml

# WE ARE ALL SET AS CROSSPLANE DID ITS WORK, LOOK HOW APPS DEFINITION CREATED SERVICE AND INGRESS
#kubectl --namespace a-team get appclaims
#NAME             HOST                              SYNCED   READY   CONNECTION-SECRET   AGE
#devops-toolkit   devops-toolkit.127.0.0.1.nip.io   True     True                        7s
#nasinha@ATLLA143MD6T devops-toolkit-crossplane % kubectl --namespace a-team \
#    get all,ingresses
#NAME                                 READY   STATUS             RESTARTS   AGE
#pod/devops-toolkit-d8fc795f9-75wxn   0/1     ImagePullBackOff   0          15s

#NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
#service/devops-toolkit   ClusterIP   10.100.51.80   <none>        80/TCP    15s

#NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
#deployment.apps/devops-toolkit   0/1     1            0           16s
