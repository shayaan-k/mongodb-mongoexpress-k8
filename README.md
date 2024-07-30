# mongodb-mongoexpress-k8
Using K8 to deploy mongo express and mongodb pods

## Start minikube session
```minikube start --vm-driver=hyperkit```

## Create mongodb deployment file
The mongo db deployment (mongo.yaml) defines specification for what the deployment should look like.

Pods in this case need to have a port and according to the mongo docker image. The default port is 27107.
I also need a username and password for the db and for safety it is stored in the mongodb-secrets file as a base 64 encoded string.
I cannot reference a secret file without creating it first. Makes sense because it doesnt exist yet. 
I need to run ```kubectl apply -f mongo-secret.yaml```
```kubectl get secret``` will show all created secrets

With the secrets not references in mongodb.yaml I can now apply it ```kubectl apply -f mongodb.yaml```
```kubectl get all``` shows pod, deployment, and replicaset all being created.

## Create Internal Service
Adding Service in the same yaml file as deployment because they belong together.

> The kind is of type service
> selector is mongodb becuase I want this service to connect to the deployment. Selector will help the service find the pods it is attached to.
> port is the exposed port
> targetPort is the container port. It must match what i exposed in the deployment

```kubectl apply -f mongodb.yaml```
Deployment remains unchanged because nothing was modified in that config and service is created.

```kubectl get service``` shows our service running.

```kubectl describe servise mongodb-service```. Here I can see that the service is attached to the correct pod. 
```kubectl get pod -o wide```

## Mongodb Express Deployment and Service
Create a mongo-express.yaml configuration file which pulls the mongo-express docker image.
We need port and credentials
> port is 8081
> MongoDB Address is stored in MONGODB_SERVER
> Credentials is stored in ADMINUSER and ADMINPASS

We want to store the Server address in a seperate Config map so that is in in a centralized location and other applications can also use it.
The server name in the config map is simply the name of the mongodb service that we created

Just like secrets, the config map needs to be applied before it can be used by the express configuration. 
```kubectl apply -f mongodb-configmap.yaml```
```kubectl apply -f mongodb-express.yaml```
```kubectl get pod``` shows the express pod is now running
```kubectl logs EXPRESSPODNAME``` will give the logs of that pod and in the logs it shows that the server is open and the database is connected.

## Allow Mongo Express to be accessible through browser
For this we need to create a Mongo Express Service. This Service will be written in the same config file as the mongo express deployment.

Start by making the express service the same as the internal service. From this point i need to make it an external service.
To do that I must give it the type of a Load Balancer.
(Should not be confused because internal and external services are both capable of load balancing k8 people are bad with names)
Type load balancer assigns an external IP address to the pod and accepts requests.
We need a place to accept requests and so we assign a third port called nodePort. nodePort is the port of the external IP address. The range of nodePorts is 30000 to 32767

With that all done I can apply the newly created service
```kubectl apply -f mongodb-express.yaml``
```kubectl get services```
Type of mongodb-express-service is Load Balancer like we specified.
No need to define ports for internal services as they are Cluster IPs by default.

To assign the IP for the external service i need to run
```minikube service mongo-express-service```

## Creating a DB
If I now created a db form the express webpage. the following
> Request recieved from external service
> External service forwards to express pod
> pod connected to mongodb internal service
> mongodb internal service forwarded it to pod