# The template of Helm chart for microservices architecture application

## Requirements

- Kubectl
- Helm
- WM Virtualbox
- Minikube

## Installation

### Kubectl, Helm, WM VirtualBox

Links:

- https://kubernetes.io/docs/tasks/tools/install-kubectl/
- https://helm.sh/docs/using_helm/#installing-helm
- https://www.virtualbox.org/wiki/Downloads

### Minikube installtion

Links

 - https://kubernetes.io/docs/tasks/tools/install-minikube/
 - https://habr.com/company/flant/blog/333470/

#### Preparatory actions  

- Check if the local machine supports virtualization

```
$ cat /proc/cpuinfo | grep 'vmx\|svm'
flags        : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm epb tpr_shadow vnmi flexpriority ept vpid fsgsbase smep erms xsaveopt dtherm ida arat pln pts
```

If response is not empty then all is OK

- Check and set up virtualization option in BIOS (see https://helpdeskgeek.com/how-to/enable-virtualization-in-the-bios/ ).
The actual virtualization setting can be named VT-x, Intel VT-x, Virtualization Extensions, Intel Virtualization Technology, etc.
E.g. the option  'Standart CMOS Features/Intel Virtualization Technology' should be set in the value 'Enabled' (if this option is named as 'Intel Virtualization Technology')


#### Check installation  

At the first run it will be downloaded ISO image of minikube and minikube will be configurated

 - Output of 'minikube start' command:

```
  Starting local Kubernetes v1.10.0 cluster...
  Starting VM...
  Getting VM IP address...
  Moving files into cluster...
  Setting up certs...
  Connecting to cluster...
  Setting up kubeconfig...
  Starting cluster components...
  Kubectl is now configured to use the cluster.
  Loading cached images from config file.

```

 - IMPORTANT NOTICE! After successful installation of minikube and starting new cluster it needs to install Tiller (the Helm server-side component) to Kubernetes cluster on minikube. For doing it start:

 ```
  helm init
 ```
 - Check kubernetes nodes and pods

```
  kubectl get nodes

  kubectl get pods --all-namespaces
   
```

## Creating docker-registry secret and dockerconfig key:

Links:
  https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
  
Steps:

1. 
```
   docker login -u AWS -p <aws-docker-registry-token> <docker-server>
```
```
  <aws-docker-registry-token> - it should be got with command  `aws ecr get-login --no-include-email --region us-east-1`
  <docker-server> -  https://171448869871.dkr.ecr.us-east-1.amazonaws.com
```
2. 
```
   kubectl create secret docker-registry <regcred-sceret-name> --docker-server=<docker-server> --docker-username=<docker-username> --docker-password=<aws-docker-registry-token> --docker-email=<your-email> 
```

3. 
```
   kubectl get secret <regcred-sceret-name> --output=yaml > regcret.tmp.yaml
```

4. copy the value of '.dockerconfigjson' to 'dockerconfig' variable (the file secrets.yaml)

## Usage 

### Deploing

- Open the terminal, pull this repository and change directory to root directory of this repository

If the deployment doesn't exist:

```
   helm install ./ --name <deployment-name> --namespace <namespace> -f  <secrets-values.yaml> [ -f <additional.values.files.yaml>]

e.g.

   helm install ./ --name sre --namespace sre -f ./secret-values.yaml
```

If the deployment exists:

```
   helm upgrade <deployment-name> --namespace <namespace> <path-to-chart-root>   -f  <secrets-values.yaml> [ -f <additional.values.files.yaml>]

e.g.

   helm upgrade sre --namespace sre .  -f ./secret-values.yaml
```

### Deploing to minikube

```
   helm install ./ --name sre --namespace sre -f values.minikube.yaml -f ./secret-values.yaml

   helm upgrade sre --namespace sre .  -f values.minikube.yaml -f ./secret-values.yaml 

```

### IMPORTANE NOTE! 

You need add minikube addon 'ingress' for starting services in minikube if you want to request them from local machine
Start nex command:

```
   minikube addons enable ingress

```

It should be done before SRE cluster deploing in minikube

Besides you should configure /etc/hosts (for each service wich starts in minikube cluster)

```
  192.168.99.112 accounts.minikube
  192.168.99.112 sre-api-v3.minikube
  ....

```

Where 192.168.99.112  is a minikube ip address:

```
  minikube ip

```

If everything is done properly each service which had been started in minikube cluster should be available from browser by address  http://<service-name>.minikube/

E.g:

```
  http://sre-api-v3.minikube/accounts

  or 

  http://accounts.minikube/

```

### Installation Mysql in docker-machine

1. Install docker-machine: https://docs.docker.com/machine/install-machine/
2. Create docker-machine:

```
   docker-machine create --driver virtualbox dev

```
3. Create actual mysql dumps for databases from stage ( in folder <local-machine-path-to>/sre-tools/deployment/scripts/mysql/init-data on local-machine):

```

cd <local-machine-path-to>/sre-tools/deployment/scripts/mysql

mkdir init-data && cd init- data

STAGE_HOST=<stage_host>
STAGE_USER=<stage_user>
STAGE_PASWORD=<stage_password>

mysqldump -h${STAGE_HOST} -u${STAGE_USER} -p${STAGE_PASWORD} --port=3306 rccmrdb > ./rccmrdb.sql

mysqldump -h${STAGE_HOST} -u${STAGE_USER} -p${STAGE_PASWORD} --port=3306 rcimpdb > ./rcimpdb.sql

mysqldump -h${STAGE_HOST} -u${STAGE_USER} -p${STAGE_PASWORD} --port=3306 impdb > ./impdb.sql

mysqldump -h${STAGE_HOST} -u${STAGE_USER} -p${STAGE_PASWORD} --port=3306 rcsredb > ./rcsredb.sql

```
4. Run docker-machine console:

```
   docker-machine ssh
```

5. Install docker-compose: https://docs.docker.com/compose/install/  (in docker-machine virtualbox machine)

6. Change dir to <local-machine-path-to>/sre-tools/deployment/scripts/mysql (in docker-machine virtualbox machine)

6. Configure <local-machine-path-to>/sre-tools/deployment/scripts/mysql/docker-compose.yml (set passwords) and start mysql docker container (in docker-machine virtualbox machine)

```
  docker-compose up -d

```
7. Fix the script ./init-data.sh (change password according to docker-compose.yml and change dumps filenames)

8. Exec mysql docker container (in docker-machine virtualbox machine):

```
  docker exec -it $(docker ps --all | grep mysql_db | awk '{ print $1 }')

```
9. Start mysql-client and add admin priveleges to mysql user defined in docker-compose.yml as MYSQL_USER (in docker-machine virtualbox machine):

```
mysql -hlocalhost -uroot -p<MYSQL_ADMIN_PASSWORD>

> GRANT ALL PRIVILEGES ON  *.*  TO '<user>'@'<host = localhost|%>';
> FLUSH PRIVILEGES;

```
10. Start ./init-data.sh script for mysql dumps uploading (in docker-machine virtualbox machine):

```
  ./init-data.sh

```

11. For connecting to docker-machine MySQL database from local-machine (or other hosts, e.g. from minikube services):

```
  DOCKER_MACHINE_IP=$(docker-machine ip)
  DOCKER_MACHINE_MYSQL_USER=<user> /* (the same it was defined in docker-compose.yml as MYSQL_USER) */
  DOCKER_MACHINE_MYSQL_USER_PASSWORD=<password> /* (the same it was defined in docker-compose.yml as MYSQL_PASSWORD) */

  mysql -h${DOCKER_MACHINE_IP} -u${DOCKER_MACHINE_MYSQL_USER} -p{DOCKER_MACHINE_MYSQL_USER_PASSWORD} --port=3306

```

12. For connecting to docker-machine MySQL database from minikube cluster you need change MySQL DB configuration in deployment/kubernetes/helm-charts/api-v3/secret-values.yaml (the section 'dbSettings')

### Usefull commands

- Delete deployment:
```
   helm delete  <deployment-name> --purge
```

- Get pods in some namespace:
```
  kubectl get pods -n <namespace>
```

- Exec pod:
```
   kubectl exec -it <pod-name> <command> -n <namespace>>

   e.g.

   kubectl exec -it api-v3-accounts-579bdf7c45-6xs9v /bin/sh  -n sre 
```

- Show logs (from stdout) of the pod:

```
   kubectl logs -f <pod-name> -n <namespace>>

   e.g.

   kubectl logs -f api-v3-accounts-579bdf7c45-6xs9v -n sre 
```

- Delete namespace:
```
   kubectl delete --namespace <namespace>
```

- Minikube dashboard:
```
   minikube dashboard   
```

- Minikube IP address:
```
   minikube ip
```

### Configuration of services

All settings which neccassary for starting chart are in the files 'values.yaml' and 'secret-values.yaml'
The file 'secret-values.yaml' should be copied from 'secret-values.yaml.example' and filled via actual creds (it needs to request them to devops engineer).

The main settings of services which could be configured in 'values.yaml':

```
servicesSettings:
  <someService>:
    enabled: &sreApiV3Enabled true   -  should be true if  the service should be deployed 
    tag: &sreApiV3Tag sre-api-v3.v1.0.0 - the name of service image tag in docker registry
    serviceType: &sreApiV3ServiceType NodePort - provides to manage of the service type, can be 'NodePort' or 'ClusterIP' (see more information in kubernetes manual)
    ingress: &sreApiV3Ingress true - should be true if  the service should have ingress (should be open from out of cluster)
    debugMode: &sreApiV3DebugMode false - provides to start service with command 'tail -f /dev/null' for debugging

    .....
```
