# Lab 4 - Networking
## 4.1 Explore CoreDNS
![Lab 4.1 final result](../diagrams/lab_4_coredns.png)
1. Create a `public.ecr.aws/qa-wfl/qa-wfl/qakf/sbe` deployment in each of the `green` and `blue` namespaces, using the `:v2` image in `green` and the `:v1` image in `blue`.

<details><summary>show commands</summary>
<p>

```bash
kubectl create deploy lab4backend --image=public.ecr.aws/qa-wfl/qa-wfl/qakf/sbe:v1 --replicas=3 -n blue 
kubectl create deploy lab4backend --image=public.ecr.aws/qa-wfl/qa-wfl/qakf/sbe:v2 --replicas=3 -n green
```

</p>
</details>
<br/>

2. Expose them both as `clusterIP` services on `port` 80 with a `target-port` of 8080, giving them both a `name` of `backend`.

<details><summary>show commands</summary>
<p>

```bash
kubectl expose deployment lab4backend --port 80 --target-port 8080 --name backend --namespace blue
kubectl expose deployment lab4backend --port 80 --target-port 8080 --name backend -n green
```

</p>
</details>
<br/>

3. We're going to use the `busybox` image to interact with DNS using nslookups. Create a pod named `nettools` in both the `green` and `blue` namespaces. Use the `busybox` image. You'll need it to run a `command` of `sleep infinity` or it will immediately transition to a `completed` state.

<details><summary>show commands</summary>
<p>

```bash
kubectl run nettools --image=busybox -n blue --command sleep infinity
kubectl run nettools --image=busybox -n green --command sleep infinity
```

</p>
</details>
<br/>

4. Use `kubectl exec` to execute the command `nslookup backend` on the two pods in the two different namespaces.

<details><summary>show commands</summary>
<p>

```bash
kubectl exec -it nettools -n blue -- nslookup backend
kubectl exec -it nettools -n green -- nslookup backend
```

</p>
</details>
<br/>


Example output:

```
Server:         10.96.0.10
Address:        10.96.0.10:53

** server can't find backend.cluster.local: NXDOMAIN

Name:   backend.green.svc.cluster.local
Address: 10.107.29.209

** server can't find backend.svc.cluster.local: NXDOMAIN

** server can't find backend.cluster.local: NXDOMAIN


** server can't find backend.svc.cluster.local: NXDOMAIN

command terminated with exit code 1
```
<br/>

Note that CoreDNS tried a lot of variations of the name `backend` but one of the first was for `backend.green.svc.clsuter.local` in this case. When run against the prod namespace, it'll look there. This is to illustrate that CoreDNS "knows" which namespace a pod is running in and returns the appropriate lookup.

5. There's only one placeholder remaining in our simple front end application, for the data from the backing service. Let's finish that off now by `apply`ing (a copy of) the sfe deployment in both namespaces. Again, you might wish to change the `lab3frontend`s to `lab4frontend`s or simply `frontend`s.

<details><summary>show command</summary>
<p>

```bash
kubectl apply -n blue -f lab4frontend.yaml
kubectl apply -n green -f lab4frontend.yaml
```

</p>
</details>
<br/>

6. Get the new pods' details either from the dev or the prod namespaces. It doesn't matter which. Ask for `wide` output so you can see their IP addresses.

<details><summary>show command</summary>
<p>

```bash
kubectl -n production get pods --output wide
```

</p>
</details>
<br/>

Example output (modified):

```
NAME                          READY   STATUS    RESTARTS   AGE   IP               NODE
frontend-5b6dcf74cb-kvvvn     1/1     Running   0          14m   192.168.29.154   k8s-worker-0   lab4backend-676f7c57f-26cjm   1/1     Running   0          78m   192.168.230.25   k8s-worker-1   nettools                      1/1     Running   0          7m    192.168.230.30   k8s-worker-1
```

<br/>

7. **cURL** one of the frontend pods' IP addresses. You should see a v2 message in the green namespace and a v1 message in the blue namespace.

8. **Optional but maybe interesting** `exec` into one of your frontend pods and run `cat /code/app/main.py`. See (around the 25th line) it's just asking for "http://backend"? That's basically what you did earlier with the nslookups. CoreDNS still knows which namespace your workload is running in. And the lack of a port number is why we had to have the service listening on port 80 but forwarding to 8080 (which the app is listening on).

## 4.2 Install an ingress controller

![Lab 4.2 final result](../diagrams/lab_4_ingress.png)

Because we've exposed the back- and front-end services as `clusterIP`s, we can't currently reach them from "outside" the cluster (the host machine). That's fine for the backend services, but the frontend needs to be reachable. If we were running in a cloud, we could expose the two frontends with two load balancer services, or use the cloud vendor's ingress controller. We'll use a couple of ingress rules behind a single Nginx ingress controller.

9. Run the following command to install an Nginx Ingress Controller. A whole bunch of resources will be created. Helm is a package manager for Kubernetes, which we haven't covered yet, but we will, in the very next module.

```bash
helm install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

10. Check that the ingress service is running. Get a list of all services in all namespaces.

<details><summary>show command</summary>
<p>

```bash
kubectl get services --all-namespaces
```

</p>
</details>
<br/>

Example output (modified):

```
NAMESPACE       NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
green           backend                              ClusterIP      10.102.60.108    <none>        80/TCP                       14m
blue            backend                              ClusterIP      10.105.142.21    <none>        80/TCP                       19m
ingress-nginx   ingress-nginx-controller             LoadBalancer   10.107.57.60     <pending>     80:31886/TCP,443:30765/TCP   72s
ingress-nginx   ingress-nginx-controller-admission   ClusterIP      10.104.181.164   <none>        443/TCP                      72s
```

<br/>

11. Test the ingress by cURL-ing the controller service's cluster IP: `kubectl curl <controller service IP>`. You should get a 404 File Not Found error because we haven't configured any backends. That's ingress backends, not our simple backend service, and we're going to sort that out now.

## 4.3 Perform a Blue-Green Switchover

12. Expose both of the frontends in the two different namespaces. Expose them both as `clusterIP` services on `port` 80 with a `target-port` of 8080, giving them both a `name` of `frontend`.

<details><summary>show commands</summary>
<p>

```bash
kubectl expose deployment lab4frontend --port 80 --target-port 8080 --name frontend --namespace blue 
kubectl expose deployment lab4frontend --port 80 --target-port 8080 --name frontend -n green
```

</p>
</details>
<br/>

13. Check that you can reach them both by finding their IP addresses and **cURL**ing them.

<details><summary>show command</summary>
<p>

```bash
kubectl get svc -A
curl green-frontend-service-ip
curl blue-frontend-service-ip
```

</p>
</details>
<br/>

14. In the default namespace, create a new _ExternalName_ service called frontend, targeting the frontend service in the **blue** namespace

<details><summary>show command</summary>
<p>

```bash
kubectl create service externalname frontend --external-name frontend.blue.svc.cluster.local
```

</p>
</details>
<br/>

15. Create an ingress rule for the frontend. You might want to call the file `ingress.yaml`.

<details><summary>show command</summary>
<p>

ingress.yaml:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sfe-ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: frontend
            port:
              number: 80
```

</p>
</details>
<br/>

15. Create the ingress.

<details><summary>show command</summary>
<p>

```bash
kubectl create -f ingress.yaml
```

</p>
</details>
<br/>

16. Test the routing by cURL-ing the ingress controller's IP. You should see the response from the frontend returned, showing `v1` of the backend.

<details><summary>show command</summary>
<p>

```bash
kubectl curl \<ingress-ip\>
```

</p>
</details>
<br/>

17. Now we will switch the traffic from the blue environment (`v1`) to the green environment (`v2`). Update the ExternalName service so that it now targets the green namespace.

<details><summary>show command</summary>
<p>

```bash
kubectl patch svc frontend -p='{"spec": {"externalName": "frontend.green.svc.cluster.local"}}'
```

</p>
</details>
<br/>

18. cURL the ingress controller IP again. You should now see that the traffic has switched to the green namespace and the response includes `v2` from the backend.

## 4.4 Perform a Canary Rollout

19. Now we will perform a canary rollout within the green namespace. Edit the existing `backend` deployment to add another label, `track=stable`.

<details><summary>show command</summary>
<p>

```bash
kubectl -n green label deploy lab4backend track=stable
```

</p>
</details>
<br/>

20. Create a new deployment in the green namespace, which uses `v3` of the backend, and label it appropriately.

<details><summary>show command</summary>
<p>

```bash
kubectl -n green create deploy lab4backend-canary --image=public.ecr.aws/qa-wfl/qa-wfl/qakf/sbe:v3 --replicas=1
kubectl -n green label deploy lab4backend-canary app=lab4backend track=canary
```

</p>
</details>
<br/>

21. cURL the ingress controller again. You will see either `v2` or `v3` of the application. Repeat the cURL command multiple times to see the response vary. Based on the relative deployment sizes (3-to-1), you should see `v3` about one in every four responses.

<details><summary>show command</summary>
<p>

```bash
kubectl curl \<ingress-ip\>
```

</p>
</details>
<br/>

22. Now scale the canary deployment **up** by one replica, and the original deployment **down** by one replica.

<details><summary>show command</summary>
<p>

```bash
kubectl -n green scale deploy lab4backend-canary --replicas=2
kubectl -n green scale deploy lab4backend --replicas=2
```

</p>
</details>
<br/>

23. cURL the ingress IP several more times. You should now see a roughly 50/50 split of `v2` and `v3` responses.

<details><summary>show command</summary>
<p>

```bash
kubectl curl \<ingress-ip\>
```

</p>
</details>
<br/>

24. Scale the deployments again, so that the canary has three replicas, and the original only one.

<details><summary>show command</summary>
<p>

```bash
kubectl -n green scale deploy lab4backend-canary --replicas=3
kubectl -n green scale deploy lab4backend --replicas=1
```

</p>
</details>
<br/>

25. cURL the ingress again to see the distribution of responses - it should now be `v3` roughly 3 responses out of every four.

<details><summary>show command</summary>
<p>

```bash
kubectl curl \<ingress-ip\>
```

</p>
</details>
<br/>

26. Finally, scale the original deployment all the way down to zero replicas, leaving only the `v3` deployment.

<details><summary>show command</summary>
<p>

```bash
kubectl -n green scale deploy lab4backend --replicas=0
```

</p>
</details>
<br/>

27. cURL-ing the ingress IP should now return only `v3` responses.

28. That's it, you're done! Let your instructor know that you've finished the lab.