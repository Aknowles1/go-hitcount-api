
------------------------------------------------------------------------------------------------------------------------------------------
<b>Go API Practice</b>
------------------------------------------------------------------------------------------------------------------------------------------
pre-requisites:

- Docker 
- Go
- Kind
---------------------------------------------------------------------------------------

<h3>TLDR/Quickstart</h3>
<br />
See above for some further exlaination on these commands/the pattern I decided on
<br />
install kind:
https://kind.sigs.k8s.io/docs/user/quick-start/
<br />

```
 kind create cluster --config=kind/cluster-spec.yaml

 kubectl apply -f k8s-manifests/mysql-manifests/mysql-ns.yaml
 kubectl apply -f k8s-manifests/mysql-manifests/mysql-pv-pvc.yaml
 kubectl apply -f k8s-manifests/mysql-manifests/mysql-secret.yaml
 kubectl apply -f k8s-manifests/mysql-manifests/mysql.yaml

 kubectl apply -f k8s-manifests/go-api-manifests/web-ns.yaml
 kubectl apply -f k8s-manifests/go-api-manifests/mysql-secret.yaml
 kubectl apply -f k8s-manifests/go-api-manifests/go-api.yaml

 kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```
<br />

you may have to wait a little while before deploying this next manifest, as nginx may take a few seconds ot start up for the first time. To verify correct startup, run:
<br />


``` 
k get po -n ingress-nginx --watch
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-2qhhk        0/1     Completed   0          2m1s
ingress-nginx-admission-patch-dq7vv         0/1     Completed   0          2m1s
ingress-nginx-controller-84fd4ff684-g6zws   1/1     Running     0          2m1s
 ```

<br />
Output should show the ingress-nginx-controller as running before you can deploy the next manifest

<br />

```kubectl apply -f k8s-manifests/go-api-manifests/ingress.yaml ```

<br />

query API:<br />
```curl --header "Host: api.com" localhost:80/incriment```

<br />
the above curl works, as kind is configured to bind the hostport, to the container port thats running kubernetes (kind - Kubernetes-IN-Docker)
<br />
As mentioned above, the api will persist the counter upon pod deletion of either mysql or the go api itself, as mysql is backed by a persistent volume.

------------------------------------------------------------------------------------------------------------------------------------------
<b>local development instructions.</b>
------------------------------------------------------------------------------------------------------------------------------------------
<b>MYSQL:</b>

first start the mysql docker container:
docs for reference:
https://dev.mysql.com/doc/mysql-installation-excerpt/8.0/en/docker-mysql-getting-started.html

navigate to dockerfile/mysql and run: 

```MYSQL_ROOT_PASSWORD=secure docker compose up```

verify mysql is running:

``` mysql -h localhost -P 3306 --protocol=tcp -u root --password ```

------------------------------------------------------------------------------------------------------------------------------------------
running API

run the go code
``` go run . ```

curl the endpoint:
```curl localhost:8080/incriment```


------------------------------------------------------------------------------------------------------------------------------------------

Testing DB record incriment.

```USE HITCOUNT;```

```SELECT *  FROM CountAPIHits;```

------------------------------------------------------------------------------------------------------------------------------------------

API Container Image Build:

```docker build .```

```docker tag  <version> aknowles99/go-api-practice:<version>```

```docker push aknowles99/go-api-practice:<version>```

testing:

```docker run aknowles99/go-api-practice:<version>```

------------------------------------------------------------------------------------------------------------------------------------------
<br />
<h3>deploy mysql to kubernetes:</h3>
<br />

``` 
 kubectl apply -f k8s-manifests/mysql-manifests/mysql-ns.yaml
 kubectl apply -f k8s-manifests/mysql-manifests/mysql-pv-pvc.yaml
 kubectl apply -f k8s-manifests/mysql-manifests/mysql-secret.yaml
 kubectl apply -f k8s-manifests/mysql-manifests/mysql.yaml
 ```
<br />

verify pod is running:
```
❯ k get po -n mysql
NAME                    READY   STATUS    RESTARTS   AGE
mysql-fb59fb77f-ss48r   1/1     Running   0          36s
```
<br />
deploy in the above order, as if you just do a:
<br />

``` kubectl apply -f . ```

<br />
in the relevent folder you may run into some issues with things such as the secrets/pvc/pv applying before the sql deployment starts up, which would then necesitate deleting/recreating the mysql pod.
<br />
Persistance:<br />
In this instance, ive chosen to run mysql in kubernetes, backed with a persistent volume/volume claim, just in order to be able package a complete solution with no external infra dependencies. hitcount record will persist between deployments, even if the persistent volume is deleted, provided the path the persistent volume points to is the same on next deployment.
<br />
***note: this will not persist through cluster deletions, only through pod/pv/pvc deletions, again in a perfect world i would be using a managed cloud database service 
<br />
Again, in the real world, i wouldnt likely use a nodes hostpath to persist data, id use one of the sttorage options many different cloud providers provide, such as EBS/EFS: 
https://aws.amazon.com/premiumsupport/knowledge-center/eks-persistent-storage/
<br />
again though, id prefer to use a managed DB from a cloud provider most of the time.
<br />




------------------------------------------------------------------------------------------------------------------------------------

<h3>deploy the go api to kubernetes:</h3>
<br />
please ensure you have already deployed mysql first, as the deployment will go into crashloopback without it

``` 
 kubectl apply -f k8s-manifests/go-api-manifests/web-ns.yaml
 kubectl apply -f k8s-manifests/go-api-manifests/mysql-secret.yaml
 kubectl apply -f k8s-manifests/go-api-manifests/go-api.yaml
 ```
<br />

verify pod is running:
```
❯ k get po -n web
NAME                          READY   STATUS    RESTARTS   AGE
go-api-7cf448bbdf-2m48m   1/1     Running   0          43s
```

<br />
This should spin up the go-api deployment. At the moment it points to the DNS name for the mysql service, configured by an env var. Provided you do not change the service name, or namespace, this should resolve fine across any cluster it is deployed to, as the patter goes like this:
<br />

``` <service name>.<namespace it resides in>.svc.cluster.local```

--------------------------------------------------------------------------------------------------------------------

--------------------------------------------------------------------------------------------------------------------
<h3>local development:</h3>
<br />

go-api:
``` kubectl port-forward -n web deployment/go-api :8080 ```
output should be something like: 
<br />ß

``` 
 kubectl port-forward -n web deployment/go-api :8080
 Forwarding from 127.0.0.1:39227 -> 8080
 Forwarding from [::1]:39227 -> 8080`
 ```

Use the port it provides you to hit the loopback and incriment the counter:
```
  ❯ curl 127.0.0.1:<your port from proxy command above>/count
  33
  ```
<br />
if you'd ever like to examine the db for troubleshooting purposes
<br />

mysql:
``` kubectl port-forward -n mysql deployment/mysql :3306```
<br />

output should be something like: 
``` 
 kubectl port-forward -n mysql deployment/mysql :3306
 Forwarding from 127.0.0.1:43787 -> 3306
 Forwarding from [::1]:43787 -> 3306
 ```

 <br />

Use the port it provides you to connect to the db:<br />
 ```mysql --host=127.0.0.1 --user=<user> --password=<password>  -P <your port from proxy command above> --protocol=tcp```

-----------------------------------------------------------------------------------------------------------------------------

<h3>Ingress</h3>

This is where implimentations diverge a little bit. Generally cloud providers make this easy, with specific labels that can be added to provision a loadbalancer in you cloud provider of choice. For this project, i didnt want to tie it in to any specific cloud provider. Another option which i have utilised on my raspberry pi cluster is MetalLB, which runs as a deamonset, elects a leader, advertises a "fake" ip address, and looks out for ARP requests across the network, when it catches one, it forwards it onto the nodes MAC address, the advantage of this, is you can lose a node, new leader election will take place, and the "fake" ip will be retained, just requests to it will be forwarded on to a new MAC address. This means proxies dont need to be reconfigured/services dont become unavailable just because one node has gone offline.
<br />

This however seeems out of scope for this small project, so for now im just installing an nginx ingress, configured for kind:
```kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml```

<br />

apply the ingress for the go api:
```kubectl apply -f k8s-manifests/go-api-manifests/ingress.yaml ```

<br />

curl endpoint to see count increase:
```curl --header "Host: api.com" localhost:80/incriment```

--------------------------------------------------------------------------------------------------------------------------------

TODO:

- replace text file html body storage with a db record 
- ammed the edit/save funtions to write to mysql
- Seperate docker file into multi stage builf 
- Add CSS and improve html so it doesnt look like an 8 year old did it 
