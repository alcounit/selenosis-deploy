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

## Create config map from config file (yaml/json)
``` bash
 kubectl create cm selenosis-config --from-file=/path/to/browsers.json=/etc/selenosis/browsers.json -n selenosis
```

## Create kubernetes service
``` bash
 kubectl apply -f 02-service.yaml
 ```

 ## Check service status
 ```bash
kubectl get svc -n selenosis
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
selenoid-ui     LoadBalancer   10.43.48.95    <pending>     8080:32000/TCP   8h
selenosis       ClusterIP      None           <none>        <none>           8h
selenosis-app   LoadBalancer   10.43.201.60   <pending>     4444:31000/TCP   8h
 ```
If external IP is not assigned for selenosis-app and selenoid-ui use kubernetes node as access point

port <b>31000</b> is for selenosis

port <b>32000</b> for selenoid-ui 


 ## Deploy selenosis
 ``` bash
 kubectl apply -f 03-selenosis.yaml
 ```

 #Check deployment status
 ```bash
kubectl get po -n selenosis
NAME                           READY   STATUS    RESTARTS   AGE
selenoid-ui-5bcc66c78d-dj7z7   2/2     Running   0          18m
selenosis-694c76f757-5m2ws     1/1     Running   0          132m
selenosis-694c76f757-6bgwl     1/1     Running   0          132m
 ```

If you need UI for your tests perform command
## Deploy selenoid-ui
 ``` bash
 kubectl apply -f 04-selenoid-ui.yaml
 ```
