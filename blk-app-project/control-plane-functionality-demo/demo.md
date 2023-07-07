1. Install Crossplane
2. kubectl apply -f provider-kubernetes-incluster.yaml
3. Check whether the provider is in healthy state or not
4. kubectl aplly -f custom-definition.yml
5. kubectl apply -f custom-blkapp-backend.yml
6. kubectl create ns b-team
7. kubectl apply -f custom-blkappclaim.yml
8. Notice the deployments, service and ingress created automatically under namespace b-team
9. Change the replica count in custom-blkappclaim.yml to 3
10. Notice that the deployment now has 3 replicas
