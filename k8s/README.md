this is an k8s folder for the voting app

this folder include manifests for run voting app services like : vote , redis , result , worker and postgres on minikube



**the architecture :**

\- vote : receives the user votes and send it to redis service 

\- worker : reads the vote from redis and stores it in posttgressql 

\- result service reads the result from the postgres db and display it to the user 

\- the inbound connection to the vote and result is done by one ingress service 



**#namespace :**

\- all the resources in namespase named (voting-app)



**#workload :**

\- i used deployment for redis , vote , result and worker services becuse thay do not need a stable idantity 

\- the deployment creates Replecaset and the Replicaset maneges the pods 



**#postgresSQL :**

\- i used statfulset for the db becuse it need a pvc and stable idantity fot the db pod

\- i deplyed standalone db 

\- i used hedless service named "db" and ckusterIP : none 

\- the statfulset creates PVC for the db data , so if the pod have been deleted the data servived 



**#redis :**

\- redis is used as an temporary queue between vote and worker 

\- i did not used a pv for redis because the real durable data is in the postgres db


**#worker :** 

* the worker dose not have a service because it dose not accept an inbound connections 



\# configmap :
the unsensitive variables like : "DB\_HOST=db" and REDIS\_HOST=redis is in the app-config manifest file 



\#postgres secret :

* db credintals like username , passeword , database is writen inside postgres-secret and did not hrdcoded in the yamal files 



\# registry Secret:

* the vote and worker and result images is stores in private Docker Hub repository 
* k8s use registry-secret to pull the private images
* the secret is bounded to the pods used imagePullSecrets
* imgaes used :

&#x20; yamanradan/voting:vote-v1

yamanradan/voting:worker-v1

yamanradan/voting:result-v1



\# Services: 

* all the internal service use ClusterIP 
* vote and result on port 80
* redis on port 6379
* postgresSQL on port 5432
* no nodeport in the final version

\# ingress :

* i used one nginx ingress for the vote and result frontends
* the ingress uses host based routing
* vote.voting.test routes to vote service on port 80
* result.voting.test routes to result service on port 80
* the frontend services stay internal and do not use NodePort


\# network policy :

* i used network policies to limit the inbound traffic between the services
* postgres allows connections from worker and result only on port 5432
* redis allows connections from vote and worker only on port 6379
* vote and result allow inbound traffic from ingress-nginx on port 80
* egress traffic is not restricted


\# probes :

* vote and result use http liveness and readiness probes on /
* redis use redis-cli ping for liveness and readiness probes
* postgres use pg_isready for liveness and readiness probes
* worker dose not expose a port so it use an exec liveness probe to check the worker process


\# resources :

* every container has cpu and memory requests and limits
* requests are used by kubernetes for scheduling the pods
* limits control the maximum resources that the container can use


\# persistent storage :

* postgres data is stored in a PVC created by the statefulset
* the PVC use the minikube standard StorageClass
* the StorageClass dynamically creates the PV
* deleting the db pod dose not delete the PVC so the data still exists when the pod is recreated


\# deploy from scratch :

start a clean minikube cluster :

```bash
minikube delete
minikube start --cpus=4 --memory=6144 --driver=docker
minikube addons enable ingress
minikube addons enable metrics-server
```

create the namespace :

```bash
kubectl apply -f k8s/shared/namespace.yaml
```

create postgres secret from the local ignored env file :

```bash
kubectl create secret generic postgres-secret \
  --from-env-file=.secrets/postgres.env \
  -n voting-app
```

create docker hub registry secret :

```bash
read -s -p "Docker Hub token: " DOCKERHUB_TOKEN
echo

kubectl create secret docker-registry registry-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=yamanradan \
  --docker-password="$DOCKERHUB_TOKEN" \
  -n voting-app

unset DOCKERHUB_TOKEN
```

deploy all the manifests :

```bash
kubectl apply -R -f k8s/
```


\# validation :

check all the pods :

```bash
kubectl get pods -n voting-app
```

check the services and ingress :

```bash
kubectl get svc -n voting-app
kubectl get ingress -n voting-app
```

check the pvc :

```bash
kubectl get pvc -n voting-app
```

check the network policies :

```bash
kubectl get networkpolicy -n voting-app
```

the application flow was tested from vote to redis then worker then postgres and the result service reads the stored votes from postgres

postgres persistence was tested by deleting db-0 and checking the vote data again after the statefulset recreated the pod
