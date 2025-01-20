After completing the continuous integration [here](../az_devops_ci_pipelines_implementation_lab/readme.md),
For the continuous delivery we will use gitops approach.

In the previous continuous integration [part](../az_devops_ci_pipelines_implementation_lab/readme.md), once the docker image is created in the build stage, azure pipeline automatically triggers the push stage. Then the artefact is pushed into azure containers registries (ACR).

In this continuous delivery part we will use the gitops approach, and use a gitops tool called ArgoCD. With we will deploy the containers into our kubernetes cluster.

![screenshot](images/cicd.PNG)

We will add a new stage to our current pipeline. This new stage will update the new created image version to the azure repository (specific yaml files or kubernetes manifests in k8s-specifications). In the azure repo, there is a script called `updateK8sManifests.sh`. Once an image is pushed to the ACR the shell script will go through this manifests and update the value of the key `image`.
For example: - image: user-azure-ci-cd/business-app:107
Gitops is always watching for a change in the git repository, as soon as an update is made to a manifest, gitops pulls the change and deploy it to the kubernetes cluster.

We will also write the creation of kubernetes cluster and configure argocd.

# Create kubernetes kluster on azure portal with Azure Kubernetes Service (AKS)
`az portal` > `Kubernetes services` > `Create` > `Kubernetes cluster`

![screenshot](images/k8s_basics.PNG)

Select agent pool

![screenshot](images/select_agent_pool.PNG)

Update agent pool

![screenshot](images/update_node_pool.PNG)

`Review and create` > `Create` will create a kubernetes cluster.

## Login to the cluster
We suppose azure CLI is already installed and configured on your system. Enter following to login
```shell
az login
```
```shell
az aks get-credentials --name azuredevops --overwrite-existing --resource-group azurecicdt
```
You should then get the output ```Merged azuredevops as current context in <users>/.kube/config```

## Install ArgoCD to the cluster
On your local system Windows/WSL/MacOS/Linux follow the step 1 from this page
https://argo-cd.readthedocs.io/en/stable/getting_started/
These instructions will create a namespace `argocd`, and deploy all the manifest resources such as redis, the argo workloads, services, repo servers, etc... 
When the deployment is done, run following to check if all pods are up and running:
```shell
kubectl get pods -n argocd
```
Expected output:
```shell
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          5m35s
argocd-applicationset-controller-64f6bd6456-vvbdp   1/1     Running   0          5m36s
argocd-dex-server-5fdcd9df8b-v8p5p                  1/1     Running   0          5m36s
argocd-notifications-controller-778495d96f-hbkmz    1/1     Running   0          5m36s
argocd-redis-69fd8bd669-r4bv9                       1/1     Running   0          5m36s
argocd-repo-server-75567c944-grxbx                  1/1     Running   0          5m36s
argocd-server-5c768cdd96-stfbg                      1/1     Running   0          5m36s
```
All pods should be running.

## Configure ArgoCD with the kubernetes cluster
### Login to argocd
Run:
```shell
kubectl get secrets -n argocd
```
Output:
```shell
NAME                          TYPE     DATA   AGE
argocd-initial-admin-secret   Opaque   1      12m
argocd-notifications-secret   Opaque   0      13m
argocd-redis                  Opaque   1      12m
argocd-secret                 Opaque   5      13m
```
Let's copy and edit the secret `argocd-initial-admin-secret`:
```shell
kubectl edit secrets argocd-initial-admin-secret -n argocd
```
Output:
```shell
apiVersion: v1
data:
  password: <PASSWORD>
kind: Secret
metadata:
  creationTimestamp: "2025-01-20T19:27:05Z"
  name: argocd-initial-admin-secret
  namespace: argocd
  resourceVersion: "17460"
  uid: 046cb23d-ecba-4d46-a7b0-fa7f18e73c68
type: Opaque
```
Secrets are base64 encoded, so we need to decode it.
We copy the password from the previous output and decode as follow:
- on linux
```shell
echo <PASSWORD> | base64 --decode
```
- on windows 
```shell
echo TlNYZlVKdFpvbU5xM3dIRw== | base64 --decode
```

The output is the decoded password. If the output contains a percentile at the end, just ignore it.
Now we have the admin password, and the argocd admin username is `admin`.
### Access argocd 
Let's run this command to check the type of the argocd-server:
```shell
kubectl get svc -n argocd
```
Output:
```shell
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.105.178.144   <none>        7000/TCP,8080/TCP            35m
argocd-dex-server                         ClusterIP   10.104.85.223    <none>        5556/TCP,5557/TCP,5558/TCP   35m
argocd-metrics                            ClusterIP   10.105.70.139    <none>        8082/TCP                     35m
argocd-notifications-controller-metrics   ClusterIP   10.101.90.254    <none>        9001/TCP                     35m
argocd-redis                              ClusterIP   10.98.6.26       <none>        6379/TCP                     35m
argocd-repo-server                        ClusterIP   10.97.91.196     <none>        8081/TCP,8084/TCP
35m
argocd-server                             ClusterIP   10.111.248.83    <none>        80/TCP,443/TCP
```
It's running in `ClusterIP` node.
Now let's access the argocd UI, for that we need to expose argocd in the nodeport mode.
```shell
kubectl edit svc argocd-server -n argocd
```
Change `type: ClusterIP` to `type: NopePort` and save the file.
If we run again `kubectl get svc -n argocd`, `argocd-server` should now have the type `NodePort`
```shell
argocd-server                             NodePort    10.111.248.83    <none>        80:32238/TCP,443:31727/TCP   39m
```
Whereas the (http) node port is `32238`. 
Now let's get the node external IP:
```shell
kubectl get nodes -o wide
```

## Write the update shell script