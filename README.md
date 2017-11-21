# Flux demo

Clone `https://github.com/weaveworks/flux` and go to the `deploy` directory.
Edit the deployment to point to your application on GitHub
Then create all the manifests

```
$EDITOR ./deploy/flux-deployment.yaml
kubectl apply -f ./deploy
```

To be able to connect to your GitHu repo, flux will use a SSH key. You need to add the key to the repo. Use `fluxctl` to get it.
See [setup](https://github.com/weaveworks/flux/blob/master/site/standalone/setup.md#connecting-fluxctl-to-the-daemon) instructions and run `fluxctl identity` 

```
$ fluxctl identity
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ9VWRA15fue5E3dqDKsX03vsqDlRScwxtWxYVXnCHoS
```

Now in yoru Git repo, you can add all the YAML manifests that make up your application. GitOps will be in effect and every git push resulting in a commit will result in the flux daemon pulling the new version of the manifests and applying the difference in your Kubernetes cluster.

The role of sealed-secrets is to be able to store sensitive secrets  -like passwords- as encrypted Kubernetes objects and benefit from Flux at the same time. If we were to use the default Secret objet from k8s those would not be encrypted (they are only base64 encoded).

Sealed-secret are a custom object implemented thanks to a Custom Resource Definition and a controller running in your cluster

## Sealed-secrets


