# flux-kustomize-example
Sample repo to play with the folder structure for Flux


## Boostrapping
In order to bootstrap a docker-desktop cluster with a Flux installation the following must be done. 
- Start Docker Desktop Kubernetes Engine
- - You will know this is working when the application shows a green kubernetes icon saying Kubernetes engine running.
- while this is starting ensure you have the `terraform` cli installed and up to date on your system. 
    - to install on Mac
    ```shell 
    $ brew tap hashicorp/tap
    $ brew install hashicorp/tap/terraform
    $ brew update
    $ brew upgrade hashicorp/tap/terraform
    ```
- Clone down the repo
- cd `./bootstrap` and take a look at the `terraform.tfvars`
```java
host                    = "https://kubernetes.docker.internal:6443"
cluster_ca_certificate  = ""
client_certificate      = ""
client_key              = ""
```
- In order to get the following information you can run the following command: `kubectl config view --flatten --minify`
- Once you have put in the proper information to your `terraform.tfvars` run the following command: `terraform init` 
- Once  you have initiated your terraform environment run the following command `terraform plan`
- Once the plan has been complete you can execute the plan with the following command: `terraform apply --auto-approve`
- If successful you should see the following message
```diff
+ Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
``` 

## Getting Started with Flux
Download the `flux-cli` to easily manage your flux resource lifecycle tasks such as reonciliation and suspension. Follow the instructions [here](https://fluxcd.io/flux/cmd/#install-using-package-management) to install.

Once your terraform has completed the flux helm chart should be successfully installed you can check this by running `kubectl get po -n flux-system` you should see the following output: 
```shell
NAME                                           READY   STATUS    
helm-controller-77d967565f-b4wwh               1/1     Running   
image-automation-controller-65dc6fb844-s96j4   1/1     Running   
image-reflector-controller-c68896d7d-7lkkk     1/1     Running   
kustomize-controller-5575d67f8f-fn88f          1/1     Running   
notification-controller-5487f8c847-4wm5t       1/1     Running   
source-controller-69bcb7cd85-k4xhc             1/1     Running   
```
In order to use flux to manage the deployment of something you will need to do some setup first.

- Create a kubernetes secret containing your `git-credentials`
```shell
kubectl create secret generic git-creds -n flux-system --from-literal=username=<username> --from-literal=password=<yourtoken>
```
- After creating your `git-creds` secret you can then apply the `git-reposiotry.yaml`
```shell
kubectl apply -f git-repository.yaml
```
  - this will tell flux to fetch the current contents of your specified git repository to watch it for changes on your specified interval.

### Managing Deployments
Now that you have created your `kind: GitRepository` Flux will continue to watch your repo for any changes. But what does it do with that information???

Now in order to have flux deploy what you what we have to deploy another component to your cluster a `kind: Kustomization`. This next part can get a little tricky when it comes to flux. In order to specify a specific folder that we want flux to look at we have to deploy a `kind: Kustomization` as well as have a defined `Kustomization` file in our repo. 


`deployment-kustomization.yaml`
```Yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: deployment
  namespace: flux-system
spec:
  force: false
  interval: 5m
  path: ./deployment
  prune: true
  sourceRef:
    kind: GitRepository
    name: git-source
  timeout: 1m
```
`kubectl apply -f deployment-kustomize.yaml`

Here we are telling flux to go to our specified `kind: GitRepository` which we deployed earlier and to grab a specific folder called `./deployment` from within that repo.

`Kustomization`
```Yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - nginx-deployment.yaml
transformers:
  - | - 
    apiVersion: builtin
    kind: NamespaceTransformer
    metadata:
      name: notImportantHere
      namespace: flux-system
    unsetOnly: true
```
By pairing the `kind: Kustomization` with this `Kustomization` file in our repo we are telling flux in the `./deployment` folder to look specifically at the `nginx-deployment.yaml` and to apply that to the cluster.

### Managing Helm Releases
Flux follows a very similar pattern when it comes to managing Helm charts.

In order to get started on this next section you will need to change your git branch to the `desktop` branch with the following command: `git checkout desktop`

#### Creating a Helm Repo
Just how we create a `kind: GitRepository` we will now create a `kind: HelmRepository` this is much like running the `helm repo add` command on your terminal. 
```shell
kubectl apply -f ./helm/helm-kustomize.yaml
```
Now if you run the following command `kubectl get kustomization -n flux-system` you will notice that this `kind: Kustomization` is failing to reconcile...why?

If you take a closer look at this `kind: Kustomization` you will notice that we are referencing a `kind: GitRepository` called `git-dev-source` which we have not created yet.

To simulate the ability to be able to work on multiple branches simultaneously we will need to create a `kind: GitRepository` pointing at our development branch `desktop` with the following command:
```shell
kubectl apply -f git-dev-repository.yaml
```

- After deploying our `helm-kustomize.yaml` we can look at the  `Kustomization` file in our repo to see what exactly is going to be deployed. 
- After waiting a few minutes you will now see that you have a `kind: HelmRepository` and a `kind: HelmRelease` deployed onto your cluster.

Now to verify that our helm chart has successfully deployed we can run the following command: `kubectl get helmrelease -n flux-system`
