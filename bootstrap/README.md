# Bootstrap

## Configure 

Follow (configure)[../README.md] and add following variables : 
```
export FORMANCE_AWS_KEY_ID="XXX"
export FORMANCE_AWS_SECRET_KEY="XXX"

export EARTHLY_BUILD_ARGS="$EARTHLY_BUILD_ARGS,awsKeyID=$FORMANCE_AWS_KEY_ID,awsSecretKey=$FORMANCE_AWS_SECRET_KEY"

```

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