# Bootstrap

## Install component

```
earthly +deploy-all-base-components
```

## Create deployment account

Once :
```
kubectl create serviceaccount deployments
kubectl create clusterrolebinding deployments --clusterrole=cluster-admin --serviceaccount=default:deployments
kubectl create token deployments --duration=999999h # Keep the output to be used as KUBE_TOKEN variable later
```

The variable KUBE_TOKEN must be used in each local env.