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

18. Now we will use ArgoCD to automate the redeployment of the application. Begin by installing the ArgoCD control plane into your cluster

<details><summary>show command</summary>
<p>

```bash
# Download and install the ArgoCD CLI
wget https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
# Deploy ArgoCD control plane
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

</p>
</details>
<br/>

19. Wait a minute or so for the server to fully initialise, then retrieve the initial admin password for the ArgoCD installation

<details><summary>show command</summary>
<p>

```bash
argocd admin initial-password -n argocd 
```

</p>
</details>
<br/>

20. Expose the ArgoCD server via your lab VM using `kubectl port-forward`

<details><summary>show command</summary>
<p>

```bash
kubectl -n argocd port-forward service/argocd-server 8000:80 & # the & is important, to fork the proxy process into the background
\<press ENTER to regain terminal input\>
```

</p>
</details>
<br/>

21. Obtain the external IP of your VM

<details><summary>show command</summary>
<p>

```bash
curl ifconfig.io
```

</p>
</details>
<br/>

22. In a new browser tab, navigate to `https://<IP from previous step>:8000` to access the ArgoCD server. You may need to dismiss a security warning due to Argo using a self-signed SSL certificate

23. Log into ArgoCD with the username `admin` and the password you retrieved in step 19

24. From the ArgoCD dashboard, create a new application named simple-app

25. On the App's settings, enable auto-sync and self-heal

26. For the App's source, choose git and provide the URL `https://github.com/agray998/qakf-helm` - this is the source for the helm chart we worked with in part 7.1

27. For the destination, choose `in-cluster` and the `blue` namespace

28. For the values file, select `values.yaml`

29. Create the application. After a minute or so ArgoCD should autosync the configuration and deploy the resources

<details><summary>Optional Stretch Tasks</summary>
<p>

30. Create another new app in ArgoCD, named `simple-app-2`. This time set the source to point at `https://github.com/agray998/qakf-kustomize`

31. Observe that ArgoCD is also able to work with Kustomize configurations as well as Helm charts. Configure the settings on the new app to deploy the `green` overlay

32. Install Kro into your cluster

<details><summary>show command</summary>
<p>

```bash
export KRO_VERSION=$(curl -sL \
    https://api.github.com/repos/kro-run/kro/releases/latest | \
    jq -r '.tag_name | ltrimstr("v")'
)

helm install kro oci://registry.k8s.io/kro/charts/kro \
  --namespace kro-system \
  --create-namespace \
  --version=${KRO_VERSION}
```

</p>
</details>
<br/>

33. Review the provided _Resource Graph Definition_ for the simple app

<details><summary>show command</summary>
<p>

```bash
cat $HOME/qakf-labs/labs07/rgd.yaml
```

</p>
</details>
<br/>

34. Create the RGD

<details><summary>show command</summary>
<p>

```bash
kubectl apply -f $HOME/qakf-labs/labs07/rgd.yaml
```

</p>
</details>
<br/>

35. In a new browser tab, log into GitHub and fork your own copy of `https://github.com/agray998/qakf-kustomize-kro`

36. Fill in `simple-app.yaml` in each of the overlays based on the relevant configurations from earlier

37. In ArgoCD, create a third app, called simple-app-kro. Enable auto-sync, self heal AND auto-create namespace

38. Set the source to YOUR fork of the GitHub repo, the destination namespace to simple-app-kro, and choose either overlay to deploy

39. Once the new app has synced, test it by cURL-ing the frontend service in the simple-app-kro namespace

<details><summary>show command</summary>
<p>

```bash
kubectl -n simple-app-kro get svc frontend
kubectl curl \<IP from above\>
```

</p>
</details>
<br/>

</p>
</details>
<br/>

40. Congratulations - you're done with the lab. Let your instructor know that you've finished 