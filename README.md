# selenosis-deploy
selenosis kubernetes deployment

## Clone deployment files
``` bash
git clone https://github.com/alcounit/selenosis-deploy.git && cd selenosis-deploy
```

## Create namespace
``` bash
 kubectl apply -f 01-namespace.yaml
```

## Create config map from config file
Check https://github.com/alcounit/selenosis for config examples. Create your own config and create config map from it, config file can be YAML or JSON.
``` bash
 kubectl create cm selenosis-config --from-file=browsers.json=browsers.json -n selenosis
```
``` bash
 kubectl create cm selenosis-config --from-file=browsers.yaml=browsers.yaml -n selenosis
```

## Create kubernetes service
``` bash
 kubectl apply -f 02-service.yaml
 ```

  ### Check service status
 ```bash
kubectl get svc -n selenosis
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
seleniferous    ClusterIP      None           <none>        <none>           8h
selenosis       LoadBalancer   10.43.201.60   <pending>     4444:31000/TCP   8h
 ```
If external IP is not assigned for selenosis use kubernetes node as access point

<b>selenosis:</b> <br/>
http://<loadBalancerIP|nodeIP>:<4444|31000>/wd/hub



 ## Deploy selenosis
 ``` bash
 kubectl apply -f 03-selenosis.yaml
 ```

   ### Check deployment status
 ```bash
kubectl get po -n selenosis
NAME                           READY   STATUS    RESTARTS   AGE
selenosis-694c76f757-5m2ws     1/1     Running   0          132m
selenosis-694c76f757-6bgwl     1/1     Running   0          132m
 ```

If you need UI for your tests perform command
## Deploy selenoid-ui
 ``` bash
 kubectl apply -f 04-selenoid-ui.yaml
 ```

 ## Deploy selenosis hpa
 ```bash
 kubectl apply -f 05-selenosis-hpa.yaml
 ```

  ### Check deployment status
 ```bash
kubectl get po -n selenosis
NAME                           READY   STATUS    RESTARTS   AGE
selenoid-ui-5bcc66c78d-dj7z7   2/2     Running   0          18m
selenosis-694c76f757-5m2ws     1/1     Running   0          132m
selenosis-694c76f757-6bgwl     1/1     Running   0          132m
 ```

  ### Check service status
 ```bash
kubectl get svc -n selenosis
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
selenoid-ui     LoadBalancer   10.43.48.95    <pending>     8080:32000/TCP   8h
seleniferous    ClusterIP      None           <none>        <none>           8h
selenosis       LoadBalancer   10.43.201.60   <pending>     4444:31000/TCP   8h
 ```

If external IP is not assigned for selenoid-ui use kubernetes node as access point
 
<b>selenoid-ui:</b> <br/>
http://<loadBalancerIP|nodeIP>:<8080|32000>/
