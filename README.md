[Recently](https://www.weave.works/blog/gitops-operations-by-pull-request), the fine folks of Weaveworks have been advocating for a mode of operation that they call _GitOps_

This resonates true with our SRE team at Bitnami where we use Kubernetes to manage [Bitnami Cloud Hosting](https://bitnami.com/cloud) a launchpad for over 100 applications on public Clouds.

Our internal deployment pipeline relies on Git as the source of truth. This is where we declare the state of our applications. A Jenkins based setup takes this declared state and applies it to our Kubernetes clusters. This homegrown pipeline that we have at Bitnami very closely resembles what Weave described in their latest [GitOps blog](https://www.weave.works/blog/the-gitops-pipeline).

Like Weave, we believe that _GitOps_ is the future (and for some already the present) of application management. 

One of our open source project is specifically addressing a GitOps workflow. [Sealed-secrets](https://github.com/bitnami/sealed-secrets) is a Kubernetes Custom Resource Definition controller to allow you to store even sensitive information (aka secrets) in Git. Indeed what Sealed-secrets is solving is the following problem: "_I can manage all my Kubernetes config in git, except Secrets._"

In this blog we show you how [Weave Flux](https://www.weave.works/oss/flux/) can be used in conjunction with Bitnami's [Sealed secret](https://github.com/bitnami/sealed-secrets) to create a continuous deployment pipeline where all operations are git based and where the desired state of your apps is declared in your git repos including your secrets.

## Using Flux

While you can use Flux easily on [Weave cloud](https://cloud.weave.works/), you can easily try it out on your own Kubernetes clusters.

Start by cloning the Flux repo:

```
git clone https://github.com/weaveworks/flux
```
Then go to the `deploy` directory.
Edit the deployment to point to your application on GitHub. Then launch Flux on your cluster with a `kubectl apply` as shown below:

```
$EDITOR ./deploy/flux-deployment.yaml
kubectl apply -f ./deploy
```

To be able to connect to your GitHu repo, Flux will use a SSH key. You need to add the key to the repo. Use `fluxctl` to get it.
See [setup](https://github.com/weaveworks/flux/blob/master/site/standalone/setup.md#connecting-fluxctl-to-the-daemon) instructions and run `fluxctl identity` 

```
$ fluxctl identity
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ9VWRA15fue5E3dqDKsX03vsqDlRScwxtWxYVXnCHoS
```

Now in your Git repo, you can add all the YAML manifests that make up your application. GitOps will be in effect and every git push resulting in a commit will result in the Flux daemon pulling the new version of the manifests and applying the difference in your Kubernetes cluster.

The role of Sealed-secrets is to be able to store sensitive secrets  -like passwords- as encrypted Kubernetes objects and benefit from Flux at the same time. If we were to use the default Secret object from k8s those would not be encrypted (they are only base64 encoded).

Sealed-secrets are custom objects implemented thanks to a Custom Resource Definition and a controller running in your cluster

To tie Flux and Sealed-secret together you simply need to deploy Sealed-secrets in your cluster as well.

## [Sealed-secrets](https://github.com/bitnami/sealed-secrets)

To install Sealed secret in your k8s cluster you need to do the following: First determine the latest stable release, then download the client for secret encryptiong `kubeseal` , create the CRD object and finally launch the controller. This is typical process for a CRD based controller.

```
release=$(curl --silent "https://api.github.com/repos/bitnami/sealed-secrets/releases/latest" | sed -n 's/.*"tag_name": *"\([^"]*\)".*/\1/p')

# Install client-side tool into /usr/local/bin/
GOOS=$(go env GOOS)
GOARCH=$(go env GOARCH)
wget https://github.com/bitnami/sealed-secrets/releases/download/$release/kubeseal-$GOOS-$GOARCH
sudo install -m 755 kubeseal-$GOOS-$GOARCH /usr/local/bin/kubeseal

# Install SealedSecret CRD (for k8s >= 1.7)
kubectl create -f https://github.com/bitnami/sealed-secrets/releases/download/$release/sealedsecret-crd.yaml

# Install server-side controller into kube-system namespace (by default)
kubectl create -f https://github.com/bitnami/sealed-secrets/releases/download/$release/controller.yaml
```

To generate a Sealed secret object you can start from a regular k8s secret object (not stored in k8s etcd) and then use `kubeseal` to obtain a properly encrypted secret:

```
kubectl create secret generic foobar --from-literal=password=root -o json --dry-run
{
    "kind": "Secret",
    "apiVersion": "v1",
    "metadata": {
        "name": "foobar",
        "creationTimestamp": null
    },
    "data": {
        "password": "cm9vdA=="
    }
}
```

Save it in a file and encrypt it

```
kubeseal <secret.json > sealed-secret.json
```

Now this encrypted secret can be stored in Git as you wish, and once Flux deploy it automatically, the Sealed secret controller will detect it and automatically decrypt it and generate a real k8s secret object. 


This process allows you to keep using GitOps even for your sensitive information. You keep it fully encrypted in your git repo but Flux will automatically leverage the sealed secret controller to generate the valid secret object that your other manifests can use.


## [Example](https://github.com/sebgoa/flux-demo) with Mysql Database

To illustrate all of this, write a Pod manifest to run Mysql database using a `mysql` secret that holds the MySql root password:

```
apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  containers:
  - image: mysql:5.5
    name: db
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: barfoo
    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql
            key: password
  volumes:
  - name: barfoo
    persistentVolumeClaim:
      claimName: mysql
```

But instead of storing an unencrypted secret you store in Git a _Sealed-Secret_ which is safe to share publicly.

```

apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: mysql
  namespace: default
spec:
  data: AgAIrZ8LY52S5kBm7FTrL8Zo4cDDDdMzfoIoEiuYd2o3RGbHEqi9cEYgQqncyFHHXvb7NIjQ97aNkPt9N0D/0IlDF43ZF9l7OUU5aUPoNgpuNYf8cqgor5HM6rt6tHdkrRYcXJPvTUktorY7g9KOuwu7JimK0qVU/NhViZlgUvDhxIDnkwpcfj8TbXNXwBXrXi94IvbCuxeHpqi2z7xDJXVmo9ZAbLSboQooO7+MA56ZcqpkrKv0VflYEAEO2V+NpJYS3CNZh4t4wM/HAlMkW7TXnsBk325cahB5+h46uZs1Tlndjx/cIwvF9WIv0XT5EdMVkgYD2Br3vYHKgRK9Hn6qF0HnB6Xma/Ie27iyNE50uzG9QMX6X4p7NWRlKFHevQYYCsxxVoIdNYD3c4hpg7d72zkZ2hT18bqWZh5ApJTQcAhrpxUsrB0RorFn8yomcQsooVXJtZrFQDNcJopIu/J6oGYPJ/ABvk3uzjm8IfHYrHzr1qedrjErenp18qaL6SrvEc0aGQPAyseGMtka4O4ufvGHLUUx+/W/j63yAEr0Xdm+9dTcoX9cie5rbPq90zdaLSNoCRvhqcL5S5SisOZ/AM1832wZ/6jVdkBWram4B+ZfeRgMvQ4xGkKUlJpoZYTgSik9cRdH5Hmld1OQeoM4nQCUabucoG7djOQK6TY1JWzX2BhNATRPPoEYNojhk1hNSkRsjplhCTr7Xa87yf5RZ3NFHeMFNsDV8EOY8gpHmlREd5MR7jFfss7/B+KLLW+XEgKBJILJAIQbJrR3bqI/WOfdsxLT7SMGfRjLBmehiuzFsekePL+zX0jURrro3AkZqsKCcHtjFemMdyDXO4dBgkT/eRyGVh9+dnPio9tnkRRCHOlb1MY6k/jTlvog
```

If you are curious to learn more about the encyrption process please check the [details](https://github.com/bitnami/sealed-secrets#details) or a more detailed [blog](https://engineering.bitnami.com/articles/sealed-secrets.html).

The result is that in your cluster you will see a running mysql Pod that is using a Kubernetes Secret to store its root password. However the actuall password is stored safely in Git as a sealed secret.

```
kubectl get pods --all-namespaces
NAMESPACE     NAME                                        READY     STATUS                       RESTARTS   AGE
default       flux-86688864d6-5cqgl                       1/1       Running                      0          1h
default       memcached-5fd59f656f-n5cjp                  1/1       Running                      0          1h
default       mysql                                       1/1       Running 						  0          30m
kube-system   sealed-secrets-controller-cd4f586fc-xp2tx   1/1       Running                      0          36m
...
$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
default-token-nqg8d   kubernetes.io/service-account-token   3         1h
flux-git-deploy       Opaque                                2         1h
flux-token-5tvs9      kubernetes.io/service-account-token   3         1h
$ kubectl get sealedsecret
NAME      AGE
mysql     36m
```