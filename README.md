# selenosis-deploy
selenosis kubernetes deployment

Clone deployment files
``` bash
git clone https://github.com/alcounit/selenosis-deploy.git && cd selenosis-deploy
```

Create namespace
``` bash
 kubectl apply -f 01-namespace.yaml
```

Create config map from config file (yaml/json)
``` bash
 kubectl create cm selenosis-config --from-file=/path/to/browsers.json=/etc/selenosis/browsers.json -n selenosis
```

Create kubernetes service
``` bash
 kubectl apply -f 02-service.yaml
 ```

 Deploy selenosis
 ``` bash
 kubectl apply -f 03-selenosis.yaml
 ```

 Deploy NodePort service if need
 ```yaml
 apiVersion: v1
kind: Service
metadata:
  name: selenosis-app
  namespace: selenosis
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: selenium
    port: 4444
    protocol: TCP
    targetPort: 4444
  selector:
    app: selenosis
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
 ```
To get externar port on which selenosis will be accept incoming connections execute:

``` bash
kubectl get services -n selenosis
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
selenosis       ClusterIP   None           <none>        <none>           97d
selenosis-app   NodePort    10.43.5.26     <none>        4444:30206/TCP   97d
```
<b>30206</b> will be external port for communication with selenosis
