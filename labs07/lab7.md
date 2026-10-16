# Lab 7 - Kubernetes GitOps

## 7.1 Helm

1. Helm is already installed in your lab environment. In fact, we have used it already to install the NGINX Ingress Controller. Verify the installation.

<details><summary>show command</summary>
<p>

```bash
helm version
```

</p>
</details>
<br/>

2. A helm chart for the simple backend and frontend we have been working with has been downloaded into your environment in `$HOME/qakf-helm`. Review the files therein, especially `values.yaml` and the contents of the `templates` directory.

3. Begin by installing a release with the chart's default configuration into the `blue` namespace

<details><summary>show command</summary>
<p>

```bash
helm install initial-release --namespace blue $HOME/qakf-helm
```

</p>
</details>
<br/>

4. Wait a minute or so for the deployments to come up, then verify the installation by cURL-ing the frontend service

<details><summary>show command</summary>
<p>

```bash
kubectl -n blue get svc frontend
kubectl curl \<ip from above output\>
```

</p>
</details>
<br/>

5. Look at the response. Note that, absent a password provided by us, the chart has generated one randomly. Suppose we wish to change that password. We can do so by using `helm upgrade` to reconfigure the release

<details><summary>show command</summary>
<p>

```bash
# --reuse-values flag ensures that any custom values we might have previously supplied are preserved, unless we override them explicitly
helm upgrade initial-release --namespace blue --reuse-values --set frontEnd.password="someSuperSecretPassword" $HOME/qakf-helm
```

</p>
</details>
<br/>

6. Wait for the resources to be reconfigured, then cURL the frontend again. You should now see the password you set.

<details><summary>show command</summary>
<p>

```bash
kubectl -n blue get svc frontend
kubectl curl \<ip from above output\>
```

</p>
</details>
<br/>

7. Now we will deploy the `v2` version of the app into the green namespace. We are going to change several values now, so we will use a cutstom values file. Copy the default values from the chart into your working directory

<details><summary>show command</summary>
<p>

```bash
cp $HOME/qakf-helm/values.yaml .
```

</p>
</details>
<br/>

8. Open the values file in an editor, and make the following changes:  

```
frontEnd:
  replicaCount: 1
  version: v1
  colour: "green" <== colour changed from 'blue' to 'green'
  port: 8080
  password: "aNewSuperSecretPassword" # <== this line added

backEnd:
  replicaCount: 3
  version: v2 # <== version changed from 'v1' to 'v2'
  port: 8080
```
9. Create a new release in the `green` namespace, using your custom values file for the configuration

<details><summary>show command</summary>
<p>

```bash
helm install release-2 --namespace green -f ./values.yaml $HOME/qakf-helm
```

</p>
</details>
<br/>

10. Wait for the deployments to be ready, then test the new release using cURL

<details><summary>show command</summary>
<p>

```bash
kubectl -n green get svc frontend
kubectl curl \<ip from above output\>
```

</p>
</details>
<br/>

11. Clean up ready for the next part of the lab by uninstalling both releases

<details><summary>show command</summary>
<p>

```bash
helm uninstall initial-release -n blue
helm uninstall release-2 -n green
```

</p>
</details>
<br/>

## 7.2 Kustomize

12. Now, for comparison, we will use Kustomize to deploy the same set of resources. A starter configuration has been made available in `$HOME/qakf-kustomize`. Review these files. In particular, compare the base version of `be-deploy.yaml` with the two overlays. Note that the overlays only define the minimal information required to patch the attributes they wish to modify - in this case, specifying 3 replicas instead of the default of 1. All the rest of the config is unchanged from the base, and therefore does not need to be repeated. Also note the use of _transformers_ and _generators_ in the overlays' `kustomization.yaml` files - these allow for management of image tags and data objects without needing to reduplicate the object yaml. 

13. Apply the `blue` overlay to deploy `v1` of the application into the `blue` namespace

<details><summary>show command</summary>
<p>

```bash
# need to create a password file to hold the password for the secret
echo "bluePassword" >> $HOME/qakf-kustomize/overlays/blue/password
# kubectl apply -k is used to pass configs through kustomize before applying
kubectl apply -k $HOME/qakf-kustomize/overlays/blue
```

</p>
</details>
<br/>

14. As before, test the application by cURL-ing the `blue` frontend

<details><summary>show command</summary>
<p>

```bash
kubectl -n blue get svc frontend
kubectl curl \<ip from above output\>
```

</p>
</details>
<br/>

15. Now apply the `green` overlay, to deploy the application, with the `v2` backend, into the `green` namespace.

<details><summary>show command</summary>
<p>

```bash
echo "greenPassword" >> $HOME/qakf-kustomize/overlays/green/password
kubectl apply -k $HOME/qakf-kustomize/overlays/green
```

</p>
</details>
<br/>

16. And again, test by cURL-ing the `green` frontend

<details><summary>show command</summary>
<p>

```bash
kubectl -n green get svc frontend
kubectl curl \<ip from above output\>
```

</p>
</details>
<br/>

17. Clean up by deleting both sets of resources

<details><summary>show command</summary>
<p>

```bash
kubectl delete -k $HOME/qakf-kustomize/overlays/blue
kubectl delete -k $HOME/qakf-kustomize/overlays/green
```

</p>
</details>
<br/>

## 7.3 ArgoCD
<!-- TODO: Fill me in -->
