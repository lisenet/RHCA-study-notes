# Study notes Red Hat OpenShift Administration I (v3.9)
_by Tomas Nevar (tomas@lisenet.com)_

## 1. Openshift Installation
[Product Documentation for OpenShift Container Platform 3.9](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/)

Installation steps:
```
$ sudo yum install atomic-openshift-utils
$ ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
$ ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
```
Post-installation step:
```
$ oc login -u admin -p redhat --insecure-skip-tls-verify=true
$ oc get nodes --show-labels
$ oc get nodes -L region
$ ssh root@master.lab.example.com
$ oc adm policy add-cluster-role-to-user cluster-admin admin
```

## 2. Users, Roles and Projects

### Create a New User
```
$ oc create user alice
$ oc create user vince
```
Get the current list of identifies:
```
$ oc get identity
```
For the `HTPasswdIdentityProvider` module, use the `htpasswd` command:
```
$ ssh root@master
# htpasswd -b /etc/origin/master/htpasswd alice password1
# htpasswd -b /etc/origin/master/htpasswd vince password2
```
### List All Users
```
$ oc get users
```
### Remove the Capability to Create Projects for All Regular Users
Removing the `self-provisioner` cluster role from authenticated users and groups denies permissions for self-provisioning any new projects:
```
$ oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
```
Try creating a new project with a regular user, it should fail:
```
$ oc login -u alice -p password1
$ oc new-project project1
Error from server (Forbidden): You may not request a new project via this API.
```
### Create Cluster Admin
```
$ ssh root@master
$ oc adm policy add-cluster-role-to-user cluster-admin admin
```

### Create a New Project
Create two new projects:
```
$ oc new-project project1 \
  --description="Alice OpenShift Project" \
  --display-name="Playground for Alice"
```
```
$ oc new-project project2 \
  --description="Vince OpenShift Project" \
  --display-name="Playground for Vince"
```
View project details:
```
$ oc describe project/project1
Name:			project1
Created:		2 minutes ago
Labels:			<none>
Annotations:		openshift.io/description=Alice OpenShift Project
			openshift.io/display-name=Playground for Alice
			openshift.io/requester=system:admin
			openshift.io/sa.scc.mcs=s0:c13,c12
			openshift.io/sa.scc.supplemental-groups=1000180000/10000
			openshift.io/sa.scc.uid-range=1000180000/10000
Display Name:		Playground for Alice
Description:		Alice OpenShift Project
Status:			Active
Node Selector:		<none>
Quota:			<none>
Resource limits:	<none>
```

### Associate a User with a Project
To manage local policies, the following roles are available:

* **admin** - users in the role can manage all resources in a project, including granting access to other users to the project. 
* **edit** - users in the role can create, change and delete common application resources from the project. Cannot manage access permissions to the project. 
* **view** - users in the role have read access to the project.
* **self-provisioner** - users in the role can create new projects. This is a cluster role, not a project role.

Add alice as an administrator for the `project1` project:
```
$ oc policy add-role-to-user admin alice -n project1
```

Add vince a developer for the `project2` project:
```
$ oc policy add-role-to-user edit vince -n project2
```

Add alice to have read access to the `project2` project:
```
$ oc policy add-role-to-user view alice -n project2
```

### View Local Role Bindings for a Project
```
$ oc describe rolebinding.rbac -n project1
```

### Security Context Constraints (SCCs)
List security context constraints
```
$ oc get scc | awk '{ print $1 }'
NAME
anyuid
hostaccess
hostmount-anyuid
hostnetwork
nonroot
privileged
restricted
```
### Create a Service Account
Create a service account apache-account:
```
$ oc create serviceaccount apache-account
```
Associate the new service account with the `anyuid` security context:
```
$ oc adm policy add-scc-to-user anyuid -z apache-account
```
Edit the deployment config for apache:
```
$ oc edit dc/apache
```
Add the service account definition:
```
spec:
  template:
    spec:
      serviceAccountName: apache-account
```
## 3. Quota and Limits
A project can contain multiple `ResourceQuota` objects.

A `LimitRange` resource, also called a limit, defines the default, minimum, and maximum values for compute resource requests and limits for a single pod or for a single container defined inside the project.

IMPORTANT: sample resource quota and limits definitions are available in the **Red Hat OpenShift Container Platform 3.9 Developer Guide** documentation, [chapter 14](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/html/developer_guide/dev-guide-compute-resources).

### Get Quota and Limits
```
$ oc get quota
$ oc get limits
```
### Create Quota
```
$ oc create quota project1-quota --hard=memory=2Gi,cpu=200m,pods=10 -n project1
```
```
$ oc describe quota -n project1
Name:       project1-quota
Namespace:  project1
Resource    Used  Hard
--------    ----  ----
cpu         0     200m
memory      0     2Gi
pods        0     10
```
### Delete All Quota
```
$ oc delete quota --all -n project1
```
### Create Limits
File `project1-resource-limits.yaml`:
```
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "project1-resource-limits"
spec:
  limits:
    - type: "Pod"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "200m"
        memory: "16Mi"
    - type: "Container"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "100m"
        memory: "8Mi"
      default:
        cpu: "300m"
        memory: "200Mi"
      defaultRequest:
        cpu: "200m"
        memory: "100Mi"
      maxLimitRequestRatio:
        cpu: "10"
```
```
$ oc create -f project1-resource-limits.yaml -n project1
```
```
$ oc describe limits -n project1
Name:       project1-resource-limits
Namespace:  project1
Type        Resource  Min   Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---   ---  ---------------  -------------  -----------------------
Pod         cpu       200m  2    -                -              -
Pod         memory    16Mi  1Gi  -                -              -
Container   cpu       100m  2    200m             300m           10
Container   memory    8Mi   1Gi  100Mi            200Mi          -
```
## 4. OpenShift Configuration Misc

### List Nodes Including Labels
```
$ oc get nodes --show-labels
```
### Label a Node
```
$ oc label node node1.lab.example.com region=infra --overwrite=true
$ oc label node node2.lab.example.com region=apps --overwrite=true
```

### Docker Registry
Check if the registry service is already present:
```
$ oc adm registry --dry-run
Docker registry "docker-registry" service exists
```
```
$ oc get dc -n default
NAME              REVISION   DESIRED   CURRENT   TRIGGERED BY
docker-registry   1          2         2         config
registry-cosole   1          1         1         config
router            1          2         2         config
```
Get the image of the registry:
```
$ oc describe dc/docker-registry -n default | grep Image
    Image:	registry.lab.example.com/openshift3/ose-docker-registry:v3.9.14
```
### Delete Docker Registry
```
$ oc delete all -l docker-registry -n default
$ oc delete serviceaccounts/registry -n default
$ oc delete clusterrolebindings/registry-registry-role -n default
```
### Create a Docker Registry from Image
```
$ oc adm registry --images=registry.lab.example.com/openshift3/ose-docker-registry:v3.9.14 \
  --replicas=2 --selector='region=infra' -n default
```
### Browse Docker Registry
List all images in the registry:
```
$ docker-registry-cli registry.lab.example.com list all ssl
```
If you don't have `docker-registry-cli` installed, you can follow the instructions below:
```
$ sudo yum install python-flask python-requests 
$ git clone https://github.com/vivekjuneja/docker_registry_cli
$ cd docker_registry_cli
$ python ./browser.py registry.lab.example.com list all ssl
```

### Configure Persistent Storage for the Registry

IMPORTANT: examples showing the use of existing persistent volume claims are available in the **Red Hat OpenShift Container Platform 3.9 Installation and Configuration** documentation, [chapter 3](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/html/installation_and_configuration/setting-up-the-registry#storage-for-the-registry).

```
$ oc volume dc/docker-registry --add --name=registry-storage \
  -t pvc --overwrite --claim-name=<pvc_name>
```
```
$ oc describe dc/docker-registry | grep -A4 Volumes
  Volumes:
   registry-storage:
    Type:	PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:	registry-claim
    ReadOnly:	false
```

### Create a Router from Image
```
$ oc adm router --images=registry.lab.example.com/openshift3/ose-haproxy-router:v3.9.14 \
  --replicas=2 --selector='region=infra' -n default
```

### Configure Wildcard Routes
IMPORTANT: wildcard routes are described in the **Red Hat OpenShift Container Platform 3.9 Installation and Configuration** documentation, [chapter 4](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/html/installation_and_configuration/setting-up-a-router#using-wildcard-routes).

```
$ oc scale dc/router --replicas=0
$ oc set env dc/router ROUTER_ALLOW_WILDCARD_ROUTES=true
$ oc scale dc/router --replicas=2
```
Verify:
```
$ oc get dc/router -o yaml | grep -i wildcard
        - name: ROUTER_ALLOW_WILDCARD_ROUTES
```
### Default Routing Subdomain
The default routing subdomain is defined in the `/etc/origin/master/master-config.yaml` file by default.
```
$ ssh root@master
# find / -name master-config.yaml -type f -print -exec grep subdomain {} \;
```

### Add a Node Selector to a Deployment Configuration
IMPORTANT: sample template for a `nodeSelector` is available in the **Red Hat OpenShift Container Platform 3.9 Installation and Configuration** documentation, [chapter 4](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/html/installation_and_configuration/setting-up-a-router#adding-nodeselector-to-a-deployment).
```
$ oc edit dc/router
```
Add the following:
```
spec:
  nodeSelector:
    region: infra
```
### Make Node Schedulable
```
$ oc adm manage-node --schedulable=true node1.lab.example.com
```
### Unschedule and Drain Node
```
$ oc adm manage-node --schedulable=false node2.lab.example.com
$ oc adm drain node2.lab.example.com --delete-local-data
```
### Edge Secured Route
IMPORTANT: example commands are available in the **Red Hat OpenShift Container Platform 3.9 Installation and Configuration** documentation, [chapter 4](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/html/installation_and_configuration/setting-up-a-router#using-secured-routes).

Create a private key, CSR and certificate.

```
$ openssl genrsa -out php.key 2048
```
```
$ openssl req -new -key php.key -out php.csr  \
  -subj "/C=GB/ST=London/L=London/O=IT/OU=IT/CN=www.example.com"
```
```
$ openssl x509 -req -days 366 -in php.csr  \
      -signkey php.key -out php.crt
```
Generate a route using the above certificate and key:
```
$ oc get svc
$ oc create route edge --service=my-php-service \
    --hostname=www.example.com \
    --key=php.key --cert=php.crt \
    --insecure-policy=Redirect
```
### Create a Secret
```
$ oc create secret generic secret_name \
  --from-literal=key1=secret1 \
  --from-literal=key2=secret2
```

## 5. Persistent Storage Using NFS

### Configure NFS Exports
Note the following requirements for NFS exports:

* folder must be owned by the `nfsnobody` user and group.
* folder must have `rwx------` permissions (expressed as 0700 using octal).
* folder must be exported using the `all_squash` option.
```
$ ssh root@services
# mkdir -p -m0700 /srv/nfs_pv1
# chown -R nfsnobody: /srv/nfs_pv1
# echo "/srv/nfs_pv1 192.168.99.0/24(async,rw,all_squash)" >> /etc/export
# exportfs -rv
```
Be aware of the following booleans:
```
$ sudo setsebool -P virt_use_nfs=true
$ sudo setsebool -P virt_sandbox_use_nfs=true
```
### Provision NFS Volume
IMPORTANT: sample PV object definition using NFS is available in the **Red Hat OpenShift Container Platform 3.9 Installation and Configuration** documentation, [chapter 24](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/html/installation_and_configuration/configuring-persistent-storage#install-config-persistent-storage-persistent-storage-nfs).

Create an object definition file `nfs-volume.yaml` for the PV:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-volume1
spec:
  capacity:
    storage: 40Gi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /srv/nfs_pv1
    server: services.lab.example.com
  persistentVolumeReclaimPolicy: Retain
```
Create the PV:
```
$ oc create -f nfs-volume.yaml
$ oc get pv
```
Create a PVC which binds to the new PV. Create a file `nfs-claim.yaml`:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-claim1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 40Gi
```
Create the PVC:
```
$ oc create -f nfs-claim.yaml
```
### Modify the Existing Deployment Configuration
```
$ oc set volume dc/docker-registry \
  --add --overwrite --name=nfs-volume1 \
  -t pvc \
  --claim-name=nfs-claim1 \
  --claim-size=40Gi \
  --claim-mode='ReadWriteMany'
```
Check the help page for other options available.
```
$ oc set volume --help
```
## 6. Metrics Subsystem

### Search for Images
Verify that the container images required by the metrics subsystem are in the private registry:
```
$ docker-registry-cli registry.lab.example.com list all ssl | grep metrics
19) Name: openshift3/ose-metrics-cassandra
47) Name: openshift3/ose-metrics-heapster
58) Name: openshift3/ose-metrics-hawkular-metrics
```
```
$ docker-registry-cli registry.lab.example.com list all ssl | grep recycler
32) Name: openshift3/ose-recycler
```
### Create PV for the NFS Share
Create a file `metrics-pv.yml`:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: metrics
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /exports/metrics
    server: services.lab.example.com
  persistentVolumeReclaimPolicy: Recycle
```
```
$ oc create -f metrics-pv.yml
$ oc get pv
```
### Ansible Inventory File
Ansible variables for the deployment of the metrics subsystem:
```
[OSEv3:vars]
# Metrics Variables
openshift_metrics_install_metrics=True
openshift_metrics_image_prefix=registry.lab.example.com/openshift3/ose-
openshift_metrics_image_version=v3.9
openshift_metrics_heapster_requests_memory=300M
openshift_metrics_hawkular_requests_memory=750M
openshift_metrics_cassandra_requests_memory=750M
openshift_metrics_cassandra_storage_type=pv
openshift_metrics_cassandra_pvc_size=5Gi
openshift_metrics_cassandra_pvc_prefix=metrics
```
Run Ansible playbook to install metrics.

### Verify Installation
```
$ oc get pvc -n openshift-infra
$ oc get pod -n openshift-infra
$ oc get routes -n openshift-infra
```

## 7. Image Streams
An image stream comprises any number of container images identified by tags.

### Import Image into OpenShift

```
$ oc import-image ishttpd \
  --from=192.168.99.1:5000/my-httpd:1.2 \
  --confirm \
  --insecure=true \
  -n isproject
```
```
$ oc get is
NAME      DOCKER REPO                         TAGS      UPDATED
ishttpd   172.30.1.1:5000/isproject/ishttpd   latest    5 seconds ago
```

### Import Template into OpenShift
```
$ oc apply -f httpd.yml -n openshift
```

## 8. Monitoring Applications with Probes 
IMPORTANT: examples of readiness and liveness checks are available in the **Red Hat OpenShift Container Platform 3.9 Developer Guide** documentation, [chapter 34](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.9/html/developer_guide/dev-guide-application-health).

```
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  timeoutSeconds: 1
```
```
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 15
  timeoutSeconds: 1
```
```
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/health
  initialDelaySeconds: 15
```
Test:
```
$ curl http://probe.apps.lab.example.com/health
OK
$ curl http://probe.apps.lab.example.com/ready
READY
```

## 9. Docker
You need to install Docker on the host.

### Deploy a Local Registry Server
Customise the storage location:
```
$ docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /mnt/images/registry:/var/lib/registry \
  registry:2
```
### Allow Insecure Registry
Add this line to `/etc/docker/daemon.json`:
```
{ "insecure-registries":["192.168.99.1:5000"] }
```
Restart docker service:
```
$ sudo systemctl restart docker
```
### Push Image to a Local Registry
Docker client can now pull from a public registry and push to a local registry using its IP address:
```
$ docker pull centos:7
$ docker tag centos:7 192.168.99.1:5000/my-centos7
$ docker push 192.168.99.1:5000/my-centos7
$ docker pull 192.168.99.1:5000/my-centos7
```
Query local registry:
```
$ docker-registry-cli 192.168.99.1:5000 list all

-----------
1) Name: my-centos7
Tags: latest	

1 images found !
```
### Build Docker Image from a Dockerfile
Sample `Dockerfile`:
```
FROM centos:7

RUN yum -y --setopt=tsflags=nodocs install httpd && \
    yum clean all && \
    rm -rf /run/httpd/* /tmp/httpd* && \
    echo "Apache in a Docker container" > /var/www/html/index.html

EXPOSE 80

CMD ["/usr/sbin/apachectl", "-D", "FOREGROUND" ]
```
Build the image:
```
$ docker build -t my-httpd:1.0 .
```
Tag the image and push it to local repository:
```
$ docker tag my-httpd:1.0 192.168.99.1:5000/my-httpd
$ docker push 192.168.99.1:5000/my-httpd
```
Run the container:
```
$ docker run -d -p 80:80 my-httpd:1.0
```
```
$ curl http://localhost
Apache in a Docker container
```
### Create an Image From a Container
To preserve the content of a running container, we can save the container as an image so we can make other containers based on this one.
```
$ docker commit <container_id>
```
The new image will not have a repository or tag.

### Log in to an OpenShift Internal Registry

```
$ docker login -p $(oc whoami -t) -u any docker-registry-default.192.168.99.107.nip.io
```
If there is a certificate error:
```
Error response from daemon: Get https://docker-registry-default.192.168.99.107.nip.io/v2/: x509: certificate is valid for *.router.default.svc.cluster.local, router.default.svc.cluster.local, not docker-registry-default.192.168.99.107.nip.io
```
Add a record for insecure registry to `/etc/docker/daemon.json`, e.g.:
```
{ "insecure-registries":["docker-registry-default.192.168.99.107.nip.io"] }
```
Restart the service and try to log in again:
```
$ sudo systemctl restart docker
$ docker login -p $(oc whoami -t) -u any docker-registry-default.192.168.99.107.nip.io
Login Succeeded
```
### Load Docker Image from TAR and Push to OpenShift Internal Registry
```
$ docker load -i my-httpd.tar
$ docker tag <image_id> docker-registry-default.apps.lab.example.com/test/my-httpd
$ docker login -p $(oc whoami -t) -u user docker-registry-default.apps.lab.example.com
$ docker push docker-registry-default.apps.lab.example.com/test/my-httpd
```

## 10. Minishift

Minishift installation on Linux using VirtualBox:
```
$ wget https://github.com/minishift/minishift/releases/download/v1.34.1/minishift-1.34.1-linux-amd64.tgz
$ tar xf minishift-1.34.1-linux-amd64.tgz
$ export MINISHIFT_HOME=/mnt/images/minishift
$ minishift config set vm-driver virtualbox
$ minishift config set disk-size 32G
$ minishift config set cpus 2
$ minishift config set memory 4096
$ minishift config set openshift-version v3.11.0
$ minishift config set insecure-registry 192.168.0.0/16
$ minishift config view
$ minishift start
$ oc login -u system:admin
```
```
$ minishift -h
Minishift is a command-line tool that provisions and manages single-node OpenShift clusters optimized for development workflows.

Usage:
  minishift [command]

Available Commands:
  addons      Manages Minishift add-ons.
  completion  Outputs minishift shell completion for the given shell (bash or zsh)
  config      Modifies Minishift configuration properties.
  console     Opens or displays the OpenShift Web Console URL.
  delete      Deletes the Minishift VM.
  docker-env  Sets Docker environment variables.
  help        Help about any command
  hostfolder  Manages host folders for the Minishift VM.
  image       Exports and imports container images.
  ip          Gets the IP address of the running cluster.
  logs        Gets the logs of the running OpenShift cluster.
  oc-env      Sets the path of the 'oc' binary.
  openshift   Interacts with your local OpenShift cluster.
  profile     Manages Minishift profiles.
  services    Manage Minishift services
  setup       Configures pre-requisites for Minishift on the host machine
  ssh         Log in to or run a command on a Minishift VM with SSH.
  start       Starts a local OpenShift cluster.
  status      Gets the status of the local OpenShift cluster.
  stop        Stops the running local OpenShift cluster.
  update      Updates Minishift to the latest version.
  version     Gets the version of Minishift.

Flags:
      --alsologtostderr                  log to standard error as well as files
  -h, --help                             help for minishift
      --log_backtrace_at traceLocation   when logging hits line file:N, emit a stack trace (default :0)
      --log_dir string                   If non-empty, write log files in this directory (default "")
      --logtostderr                      log to standard error instead of files
      --profile string                   Profile name (default "minishift")
      --show-libmachine-logs             Show logs from libmachine.
      --stderrthreshold severity         logs at or above this threshold go to stderr (default 2)
  -v, --v Level                          log level for V logs. Level varies from 1 to 5 (default 1).
      --vmodule moduleSpec               comma-separated list of pattern=N settings for file-filtered logging

Use "minishift [command] --help" for more information about a command.
```
### Deploy Internal Registry and Router on Minishift
```
$ oc label node localhost region=infra
```
```
$ oc adm router --images=openshift/origin-haproxy-router:v3.11.0 \
  --selector='region=infra' -n default
```
```
$ oc adm registry --images=openshift/origin-docker-registry:v3.11.0 \
  --selector='region=infra' -n default
```
```
$ minishift openshift registry
172.30.224.52:5000
```
## 11. Examples

### Create Openshift App from Local Image with Secure Edge-terminated Route
```
$ oc login -u developer
$ oc new-project webserver
$ oc new-app --name=apache --docker-image=192.168.99.1:5000/my-httpd:1.0
```
The deployment will fail because of permissions.
```
$ oc rollout cancel dc/apache
```
Create a service account:
```
$ oc login -u system:admin
$ oc create serviceaccount apache-account
$ oc adm policy add-scc-to-user anyuid -z apache-account
```

```
$ oc login -u developer
$ oc edit dc/apache
```
Configure the following:
```
spec:
  template:
    spec:
      serviceAccountName: apache-account
```
```
$ oc rollout latest dc/apache
$ oc get pods
$ oc port-forward apache-2-hgskh 1234:80
```
```
$ curl http://127.0.0.1:1234
Apache in a Docker container
```
Generate a certificate:
```
$ openssl genrsa -out apache.key 2048
$ openssl req -new -key apache.key -out apache.csr \
  -subj "/C=GB/ST=London/L=London/O=IT/OU=IT/CN=apache-webserver.192.168.99.107.nip.io"
$ openssl x509 -req -days 366 -in apache.csr -signkey apache.key -out apache.crt
```
Create a route:
```
$ oc create route edge \
  --service=apache \
  --hostname=apache-webserver.192.168.99.107.nip.io \
  --key=apache.key \
  --cert=apache.crt \
  --insecure-policy=Redirect \
  -n webserver
```
Test:
```
$ curl -k https://apache-webserver.192.168.99.107.nip.io
Apache in a Docker container
```
### Create a Database Server
Use publicly accesible OpenShift 3.9 templates on [GitHub](https://github.com/openshift/origin/tree/release-3.9/examples/db-templates)

### Create WordPress Application
Use publicly accesible OpenShift 3.9 templates on [GitHub](https://github.com/openshift/origin/tree/release-3.9/examples/wordpress)

### Create a Git Server
Use publicly accesible OpenShift 3.9 templates on [GitHub](https://github.com/openshift/origin/tree/release-3.9/examples/gitserver)

### Image Streams
Use publicly accesible OpenShift 3.9 templates on [GitHub](https://github.com/openshift/origin/tree/release-3.9/examples/image-streams)
