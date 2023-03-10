# Kubernetified Todo-App (Group 15)

## Table of Contents

* [About the Project](#about-the-project)
* [Web Application](#web-application)
* [Docker Components](#docker-components)
* [Kubernetes Components](#kubernetes-components)
* [Helm](#helm)
* [Horizontal Scaling](#horizontal-scaling)
* [Update Rollout](#update-rollout)
* [Demo](#demo)

# About the Project
This project was completed by group 15 for the course Software Containerization 2023 at Vrije Universteit Amsterdam. It is our attempt at a super basic to-do list application designed to be be easily deployed by anyone using Kubernetes and the package manager Helm. Each component of the application (frontent, api, database, redis) can be deployed using kubernetes components, Docker images and containers in the scheme outlined below. Our implementation is designed to be deployed in an Microsoft Azure environment.

## Important Note:
While originally designed as a to-do list app, we later decided to reduce the scope of the project in order to focus on the deployment challenges. Consequently, only the API supports operations that would be required by a to-do list app, while the frontend is only interfacing with the user management component of the app.

## Architecture
![Deployment diagram of the containerized to-do app in this repository](https://github.com/Xantocx/to-docker/blob/main/misc/Blueprint.png)

<!-- Contributors -->
## Contributors
- Lucas Lageweg
- Maximilian Mayer
- Jeren Olsen

# Web Application
The application in this repository consists of 4 core elements:

- Vue.js frontend (see [here](https://github.com/Xantocx/to-docker/tree/main/frontend))
- Python REST API using FastAPI (see [here](https://github.com/Xantocx/to-docker/tree/main/backend))
- PostgreSQL database (using default Docker image)
- Redis as a session cache (using default Docker image)

The frontend is hereby running hosted using a NGINX web server that serves the Vue.js application. The REST API handling frontend requests uses an uvicorn ASGI web server for Python, hosting the FastAPI backend. Both components will be hosted publically, as the client code for the frontend requires an exposed API.

PostgreSQL and Redis are only accessible within the Kubernetes cluster they are deployed on, allowing the API component to persist data.

# Docker Components
While PostgreSQL and Redis have pre-existing Docker images, we created our own Dockerfiles for the [front-](https://github.com/Xantocx/to-docker/blob/main/frontend/Dockerfile) and [backend](https://github.com/Xantocx/to-docker/blob/main/backend/Dockerfile). Using these files, you can create your own images, and host them in any registry. From here, you can be used in the [Kubernetes deployment](#kubernetes-components).

To build your own images, first clone this repository:
```bash
git clone https://github.com/Xantocx/to-docker.git
cd to-docker
```
The frontend image can be built as follows:
```bash
cd frontend

# build Docker image for frontend and tag it accordingly
sudo docker build -t <your-registry>/todo-frontend-image:latest .

# push docker image to your registry
sudo docker push <your-registry>/todo-frontend-image:latest

cd ..
```

Similarly, the API image can be created:
The frontend image can be built as follows:
```bash
cd backend

# build Docker image for frontend and tag it accordingly
sudo docker build -t <your-registry>/todo-api-image:latest .

# push docker image to your registry
sudo docker push <your-registry>/todo-api-image:latest

cd ..
```

## :warning: Important Note :warning:
Please note that we configured all files in this project to work on our specific cluster configuration. This includes the registry used to host images. Thus, you will have to adjust the image names used in the corresponding Kubernetes deployment files for the [front-](https://github.com/Xantocx/to-docker/blob/main/kubernetes/deployments/frontend-deployment.yml) and [backend](https://github.com/Xantocx/to-docker/blob/main/kubernetes/deployments/api-deployment.yml), as well as any references in the [helm charts](https://github.com/Xantocx/to-docker/tree/main/todo-app-helm) to fit your specific image tags.

If you are interested in the exact commands used for our demo, please refer to our [demo script](https://github.com/Xantocx/to-docker/blob/main/demo.sh), which includes a full list of all commands, exactly as they where featured in the demo.

# Kubernetes Components
Once you created your Docker images, you can start deploying the application. The intended way for this is the usage of the Helm package manager. For that, we refer you [here](#helm). However, if you want to manually deploy the application, we also provide the standard yaml files for Kubernetes in [this directory](https://github.com/Xantocx/to-docker/tree/main/kubernetes).

First, you will have to configure your cluster with the namespace that will contain all resources we wil create for this application. This is done as follows:
```bash
kubectl apply -f ./kubernetes/config/namespace.yml
```

Next, you have to setup two secrets, containing the username and password for the PostgreSQL database, as well as an auth key for the API:
```bash
kubectl apply -f ./kubernetes/config/config.yml
```

Now, it is time for the user access control. For this application, we defined three user types:

- **admin:** The admins are allowed to access and modify all resources everywhere on the cluster
- **dev:** The developers can access and modify all resources in the namespace "todo-app" of our app
- **operator:** The operators can read all resources that are required to monitor the current state of the application (e.g., deployments, pods) and can modify resources directly related to networking within the application (e.g., ingresses, network policies)

While the users itself have to be created manually, we define the corresponding roles, and role bindings in order to enforce their rights:
```bash
kubectl apply -f ./kubernetes/access-control/admin.yml
kubectl apply -f ./kubernetes/access-control/developer.yml
kubectl apply -f ./kubernetes/access-control/operator.yml
```

Now it is time for the most important resources: the deployments. For each of our 4 core components (frontend, api, database, and Redis) we have one deployment file. These files can be applied as follows:
```bash
kubectl apply -f ./kubernetes/deployments/database-deployment.yml
kubectl apply -f ./kubernetes/deployments/redis-deployment.yml
kubectl apply -f ./kubernetes/deployments/api-deployment.yml
kubectl apply -f ./kubernetes/deployments/frontend-deployment.yml
```

In case of the database, this will create a persistent volume, as well as a corresponding persistent volume claim, which can be used to mount the volume in the database deployment. Additionally, the file will create the deployment necessary to launch a database pod on the cluster, and a ClusterIP service required for different pods to communicate with each other. Such a service, as well as the corresponding deployment, are also created for the API, Redis, and the frontend. 

> :warning: **We use CoreDNS!** Our application assumes that containers can resolve service names in order to reach them. This requires CoreDNS, so please make sure CoreDNS is installed.

Finally, we need to make sure that the frontend and the API can be reached from a client outside the cluster. Here, we rely on the [NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/), so please make sure you have set that up! You can deploy the ingress, as well as some network policies that make sure that only the required traffic is allowed as follows:
```bash
kubectl apply -f ./kubernetes/networking/network-policies.yml
kubectl apply -f ./kubernetes/networking/ingress.yml
```

> :warning: **Set up your own domain name!** Once again, we have provided the application as we deployed it on our own cluster. This means, the domain name we used to deploy the application is hardcoded in the [ingress resource](https://github.com/Xantocx/to-docker/blob/main/kubernetes/networking/ingress.yml). Make sure to update it, as we also request a TSL certificate for it (see below).

> :warning: **Network Policies!** These network policies require a networking plugin that support them. We recommend [Calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/).

## :warning: Important Note :warning:
This application uses TLS in order to secure the connection between the cluster and any users. As a consequence, the ingress deployed above also requests a certificate at [Let's Encrypt](https://letsencrypt.org/). This requires a cluster issuer. We provide one, however, feel free to deploy your own if needed. If you want to use ours, you can do so as follows:
```bash
kubectl apply -f ./kubernetes/networking/cluster-issuer.yml
```

## Convenience Scripts
We do not suggest that you use the way described above to deploy this application. However, if you choose to do so, we provided two convenience scripts to do so to start the application, and shut it down again. They can be invoked as follows:
```bash
cd kubernetes/scripts

# deploy the application
./startup.sh

# shut the application down
./shutdown.sh

cd ../..
```

# Helm
The intended way of deploying the application is the Kubernetes package manager Helm. This allows to deploy many resources all at once in a structured manner. For this project, we created the required [Helm chart](https://github.com/Xantocx/to-docker/tree/main/todo-app-helm/todo-app), and wrapped each of the four subcomponents ([frontend](https://github.com/Xantocx/to-docker/tree/main/todo-app-helm/base-frontend), [API](https://github.com/Xantocx/to-docker/tree/main/todo-app-helm/base-api), [database](https://github.com/Xantocx/to-docker/tree/main/todo-app-helm/base-database), and [Redis](https://github.com/Xantocx/to-docker/tree/main/todo-app-helm/base-redis)) in their own charts, which we integrated as dependencies. However, before you can use them, you have to set up the following additional components:

- Updated image and domain names
- Networking Plugin (such as Calico)
- CoreDNS
- NGINX Ingress Controller
- ClusterIssuer for TSL certificates

For all of that, please refer to the notes in the previous sections on [Docker](#docker-components) and [Kubernetes](#kubernetes-components). Once this is done, you can install our application as follows using Helm:
```bash
cd  ../todo-app-helm/todo-app

# update the 4 sub-charts we created for our 4 main components
helm dependency update

cd ..

# install helm chart as "todo-app" in the namespace "todo-app"
helm install todo-app todo-app --namespace=todo-app --create-namespace
```

Uninstalling the application works similar:
```bash
helm uninstall todo-app -n todo-app
```

# Horizontal Scaling
If you want to scale a deployed version of this application by increasing the amount of replicas deployed, this can easily be done. However, due to their stateful nature, this is not possible for the database and Redis. Instead, you can only scale the stateless components, including the API and the frontend. This can be done as follows:

```bash
# scaling the frontend to 10 replicas
kubectl scale deployment frontend-deployment --replicas=10 -n todo-app

# scaling the API to 19 replicas
kubectl scale deployment api-deployment --replicas=10 -n todo-app
```

The use of a horizontal autoscaler is also possible:
```bash
# use an autoscaler to deploy between 1 and 10 frontend pods
# depending on the CPU usage
kubectl autoscale deployment frontend-deployment --cpu-percent=50 \
                                                 --min=1 --max=10

# use an autoscaler to deploy between 1 and 10 api pods
# depending on the CPU usage
kubectl autoscale deployment api-deployment --cpu-percent=50 \
                                            --min=1 --max=10
```

# Update Rollout
Rolling out updates is an important part of developing a cloud-native application. For this project, two options are available.

## 1. Traditional Rollout
Assuming some code was changed in the frontend. Then this change could be deployed as follows:
```bash
# create new frontend image and push it to registry
cd frontend
sudo docker build -t <your-registry>/todo-frontend-image:new-version .
sudo docker push <your-registry>/todo-frontend-image:new-version
cd ..

# update image reference in deployment and record change in history
kubectl set image deployment frontend-deployment \
    frontend-container=<your-registry>/todo-frontend-image:new-version \
    -n todo-app --record

# review history
kubectl rollout history deployment frontend-deployment -n todo-app
```

Similarly, the API code can be re-deployed after an update:
```bash
# create new frontend image and push it to registry
cd backend
sudo docker build -t <your-registry>/todo-api-image:new-version .
sudo docker push <your-registry>/todo-api-image:new-version
cd ..

# update image reference in deployment and record change in history
kubectl set image deployment api-deployment \
    api-container=<your-registry>/todo-api-image:new-version \
    -n todo-app --record

# review history
kubectl rollout history deployment api-deployment -n todo-app
```

Finally, if necessary, these rollouts can be reverted as follows:
```bash
# revert frontend rollout
kubectl rollout undo deployment/frontend-deployment \
    --revision <REVISION_NUMBER> -n todo-app

# revert API rollout
kubectl rollout undo deployment/api-deployment \
    --revision <REVISION_NUMBER> -n todo-app
```

## 2. Canary Deployment
Alternatively, you can perform a canary deployment. That means you update the deployment only partly, and only fully update, once the change was tested in practice. This is a very useful deployment strategy, and can be implemented as follows (for simplicity, we will limit this example to the frontend component):
```bash
# deployment file updating our frontend image version from v1 to v2
kubectl apply -f ./kubernetes/deployments/frontend-canary-deployment.yml

# scale updated deployment up to 3 replicas, and the original one down to 7
# this means 30% of all traffic will receive the update
kubectl scale deployment frontend-canary-deployment --replicas=3 -n todo-app
kubectl scale deployment frontend-deployment --replicas=7 -n todo-app

# IF UPDATE SUCCESSFULLY TESTED:
# scale new deployment up to 10, meaning the full capacity
kubectl scale deployment frontend-canary-deployment --replicas=10 -n todo-app
# delete old deployment completely
kubectl delete deployment frontend-deployment -n todo-app
```

> :warning: **Note on Helm Deployment:** If you perform this type of canary deployment on an instance of the application that was launched using Helm, don't forget that the "frontend-canary-deployment" is not part of the original Helm deployment and will not be deleted when you uninstall the Helm release. In that case, you will have to delete this deployment manually.

# Demo
For the submission of this project, we were required to perform a demo during a presentation. For the purpose of this demo, we used [doitlive](https://doitlive.readthedocs.io/en/stable/), a tool for scripted live demonstrations. The script used for this demo, including all the original commands, can be found [here](https://github.com/Xantocx/to-docker/blob/main/demo.sh). Feel free to check it out, and adapt it for yourself if necessary!