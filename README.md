# Project Name : node-api

### Dockerization :
Application is Dockerized on "node:16-alpine" base image and locally tested by creating a docker-compose with created Dockerfile and PostgreSQL database.

App is tested with Postman by sending {"dish": "Manti","country": "Turkey"} and {"dish": "Sushi","country": "Japan"} as a POST request. After refresh of the 
/food endpoint, new entries appeared with "id: 3" and "id: 4".

### Continuous Integration Logic :

Two branches are chosen. Branch is "master", branch pattern is "dev". dev branch is for development environment while master is related with production.

When a new code is pushed/merged to either of these branches, pipeline should be triggered, image should be built, tagged and pushed to related registry with two tags, one of them has unique name (includes ${CI_COMMIT_BRANCH}.${CI_PIPELINE_ID}) and other one is <environment>-latest. 
But in deployment step, the environment should be different for them. When we speak on the same cluster, they should use different namespaces so one of them should be deployed to dev namespace while other one is deployed to prod. 

### Helm Deployment :

Kubernetes components such as deployment, service and hpa are templated and deployed to cluster via Helm.
For different environments, different variables are defined in values.yaml to be able to accomplish different environment requirements.
For example, in our dev environment, we may want to have less replica, less resource consumption, different database endpoints etc. To enable this, %ENVIRONMENT% parameter is used in template yamls. With a basic *sed* command, we can exchange this parameter with related environment, so related environment's value set will be read from values.yaml. When we decide to create third environment for this api, that environment's value set should be created inside values.yaml and environment name should be exchanged accordingly during deployment. 

Second *sed* command is for %IMAGETAG% parameter in Helm deployment.yaml. As mentioned above, images are tagged twice and one of them is unique. To be able 
to apply unique one to the environment, %IMAGETAG% parameter was put and this parameter will be substituted during deploy phase.

When we look at the values.yaml, there is a common value set as well. This values are selected as common because they are mostly the same in all service 
environments. AppPort, healthcheckPath, RegistryName are some of them.

#### How it works?

1 : sed -i "s/%ENVIRONMENT%/environment_name/g" * (inside the templates directory) : This command should be run after repository is pulled. This step can 
easily be automated through tools like Ansible, Jenkins, Gitlab CI and Bitbucket Pipelines. This command is important for environment distinction to be able 
to read correct values for the correct environment.

2 : helm template node-api ./helm/ --dry-run (on main directory) : dry-run the template outputs to see if is there anything wrong.

node-api : release name

/helm/ : helm directory inside project.

helm template node-api ./helm/ (on main directory) : template the project with environment related & common value sets.

helm install node-api ./helm/ (on main directory) : Installing the chart/app to Kubernetes Cluster. (creation)

helm upgrade node-api ./helm/ (on main directory) : Upgrading previously deployed helm chart.

### Local Environment Setup :

Minikube : https://computingforgeeks.com/how-to-install-minikube-on-ubuntu-debian-linux/

VirtualBox : https://computingforgeeks.com/install-virtualbox-on-ubuntu-linux/

Docker : https://docs.docker.com/engine/install/ubuntu/

### Container Configuration Change in Realtime :

Since the service itself is reading variables like DB_HOST, DB_PORT, PORT, DATABASE_PASSWORD, DB_DATABASE, DB_USER from runtime environment, they are defined in deployment.yaml as environment variables. We can change DB related environment variables in deployment.yaml, or update the values.yaml to make it permanent and template & upgrade the deployment with the same image.

Second approach is creating configmap for environment variables and read from there instead of values.yaml, Update the configmap and rollout restart the deployment.

### Steps to Create Environment and Deploy the application:

- Install minikube

- install VirtualBox (virtualization for minikube)

- Create Namespaces for dev & prod (yamls are under )

- Apply postgresql YAML files for both dev & prod accordingly (PV - PVC - secret - StatefulSet - Service order would be better)

- Connect the database server to test the connection & Apply app/queries/food.sql to database manually

- Necessary config changes : There is a value named RegistryName in values.yaml and DOCKER_REGISTRY, DOCKER_REGISTRY_USER, DOCKER_REGISTRY_PASSWORD variables should be modified accordingly.

- Run *sed* commands for each dev environment *helm/templates* directory; 
    
    sed -i "s/%ENVIRONMENT%/dev/g" * && sed -i "s/%IMAGETAG%/dev-test/g" deployment.yaml

- After *sed* command is done, dry-run the template to make sure everything is okay. Then template it and install it to the minikube k8s cluster for the first time.
    
    helm template node-api-dev ./helm/ --dry-run

    helm template node-api-dev ./helm/

    helm install node-api-dev ./helm/

- After installation finished, test api service by exposing it via minikube, get the URL with the command below, paste it browser with /food path added.

- minikube service node-api-svc --url

- After everything is done and checked, we can add, remove or update the datasets via HTTP requests.

- **Repeat the same process for prod environment** 

- This can be a little bit tricky because helm template files should be reverted to their old versions before running sed commands on them again. In automation, this will be no problem because during code fetch in pipelines, SCM mechanism will revert them back. 

- Run *sed* commands for each dev environment *helm/templates* directory; 
    
    sed -i "s/%ENVIRONMENT%/dev/g" * && sed -i "s/%IMAGETAG%/dev-test/g" deployment.yaml

- After *sed* command is done, dry-run the template to make sure everything is okay. Then template it and install it to the minikube k8s cluster for the first time.
    
    helm template node-api-dev ./helm/ --dry-run

    helm template node-api-dev ./helm/

    helm install node-api-dev ./helm/

- After installation finished, test api service by exposing it via minikube, get the URL with the command below, paste it browser with /food path added.

- minikube service node-api-svc --url

- After everything is done and checked, we can add, remove or update the datasets via HTTP requests.

Our pipeline is ready, services are deployed to both environment. We can use our pipelines by pushing code to dev and master branches. In deploy phase through pipeline, we will use *helm upgrade* instead of *helm install* because pipelines will work on existing (deployed) helm projects.