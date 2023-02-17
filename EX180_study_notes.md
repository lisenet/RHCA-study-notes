# Study notes for EX180 Red Hat Certified Specialist in Containers and Kubernetes
_by Tomas Nevar (tomas@lisenet.com)_

## Table of Contents

* [Exam Objectives](#exam-objectives)
* [Setup Homelab: Install Podman and OpenShift](#setup-homelab-install-podman-and-openshift)
* [Using Podman to Build Images and Run Containers](#using-podman-to-build-images-and-run-containers)
    * [Podman Task 1: Containerise a JBoss Application](#podman-task-1-containerise-a-jboss-application)
    * [Solution to Podman Task 1](#solution-to-podman-task-1)
    * [Podman Task 2: Build a JBoss Container Image and Run a Container](#podman-task-2-build-a-jboss-container-image-and-run-a-container)
    * [Solution to Podman Task 2](#solution-to-podman-task-2)
    * [Podman Task 3: Save Container Image](#podman-task-3-save-container-image)
    * [Solution to Podman Task 3](#solution-to-podman-task-3)
    * [Podman Task 4: Push and Pull Images from Registries](#podman-task-4-push-and-pull-images-from-registries)
    * [Solution to Podman Task 4](#solution-to-podman-task-4)
    * [Podman Task 5: Deploy a WordPress Application](#podman-task-5-deploy-a-wordpress-application)
    * [Solution to Podman Task 5](#solution-to-podman-task-5)
* [Creating Applications in OpenShift](#creating-applications-in-openshift)
    * [OpenShift Task 1: Deploy a MySQL Application using OpenShift Templates](#openshift-task-1-deploy-a-mysql-application-using-openshift-templates)
    * [Solution to OpenShift Task 1](#solution-to-openshift-task-1)
    * [OpenShift Task 2: Deploy a MySQL Application from a Template File](#openshift-task-2-deploy-a-mysql-application-from-a-template-file)
    * [Solution to OpenShift Task 2](#solution-to-openshift-task-2)
    * [OpenShift Task 3: Deploy a PHP Application using Source-to-Image (S2I)](#openshift-task-3-deploy-a-php-application-using-source-to-image-s2i)
    * [Solution to OpenShift Task 3](#solution-to-openshift-task-3)
    * [OpenShift Task 4: Deploy an HTTP Application from Image](#openshift-task-4-deploy-an-http-application-from-image)
    * [Solution to OpenShift Task 4](#solution-to-openshift-task-4)

## Exam Objectives

* Implement images using Podman
    * Understand and use FROM (the concept of a base image) instruction.
    * Understand and use RUN instruction.
    * Understand and use ADD instruction.
    * Understand and use COPY instruction.
    * Understand the difference between ADD and COPY instructions.
    * Understand and use WORKDIR and USER instructions.
    * Understand security-related topics.
    * Understand the differences and applicability of CMD vs. ENTRYPOINT instructions.
    * Understand ENTRYPOINT instruction with param.
    * Understand when and how to expose ports from a Docker file.
    * Understand and use environment variables inside images.
    * Understand ENV instruction.
    * Understand container volume.
    * Mount a host directory as a data volume.
    * Understand security and permissions requirements related to this approach.
    * Understand lifecycle and cleanup requirements of this approach.
* Manage images
    * Understand private registry security.
    * Interact with many different registries.
    * Understand and use image tags.
    * Push and pull images from and to registries.
    * Back up an image with its layers and meta data vs. backup a container state.
* Run containers locally using Podman
    * Get container logs.
    * Listen to container events on the container host.
    * Use Podman inspect.
* Basic OpenShift knowledge
* Creating applications in OpenShift
   * Create, manage and delete projects from a template, from source code, and from an image.
   * Customise catalog template parameters.
   * Specifying environment parameters.
   * Expose public applications.
* Troubleshoot applications in OpenShift
   * Understand the description of application resources.
   * Get application logs.
   * Inspect running applications.
   * Connecting to containers running in a pod.
   * Copy resources to/from containers running in a pod.


## Setup Homelab: Install Podman and OpenShift

We are going to use a server to install Podman, set up Qemu-KVM hypervisor and deploy OpenShift using CodeReady containers. We will then use the environment to prepare for the EX180 exam.

OpenShift VM needs 4 CPUs and 9GB of RAM to run.

Install Podman:

```
sudo yum install -y podman
```

KVM is ubiquitous in a way that its built into Linux and requires virtually no effort to set it up. KVM lets you turn Linux into a type 1 bare-metal hypervisor. Install the necessary packages:

```
sudo yum install -y qemu-kvm libvirt
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,qemu ${USER}
```

Install CodeReady containers (OpenShift):

Download the pull secret from the Pull Secret section of the CRC page on the Red Hat Hybrid Cloud Console and save it as `pull-secret.json`.

Download the latest release of CRC for your platform.

```
cd /tmp
curl -sSL -o crc-linux-amd64.tar.xz https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz
tar xvf ./crc-linux-amd64.tar.xz
rm -f ./crc-linux-amd64.tar.xz
sudo cp ./crc-linux-*-amd64/crc /usr/local/bin/
```

In this case we are going to use OpenShift 4.11 which is based on Kubernetes 1.24.

Configure CodeReady containers:

```
crc config set cpus 4
crc config set memory 10240
crc config set disk-size 31
crc config set kubeadmin-password changeme
crc config set consent-telemetry no
crc config set enable-cluster-monitoring no
crc config set disable-update-check yes
```

CRC will refuse to run as root, therefore you have to start the process as a regular user.

Make sure that your `/proc` filesystem is not mounted with hidepid=2 option. This option hides the `/proc` from your shell that uses polkit.

```
crc setup
crc start --pull-secret-file ./pull-secret.json
```

Retrieve the password for the developer and kubeadmin users:

```
crc console --credentials
To login as a regular user, run 'oc login -u developer -p developer https://api.crc.testing:6443'.
To login as an admin, run 'oc login -u kubeadmin -p changeme https://api.crc.testing:6443'
```

Use the `oc` command line interface:

```
eval $(crc oc-env)
```

Log in to OpenShift as admin:

```
oc login -u kubeadmin https://api.crc.testing:6443
```

Check the node status:

```
oc get no
NAME                 STATUS   ROLES           AGE   VERSION
crc-lgph7-master-0   Ready    master,worker   44d   v1.24.0+3882f8f
```

## Using Podman to Build Images and Run Containers

### Podman Task 1: Containerise a JBoss Application

Write a **Dockerfile** to containerise a JBoss EAP 7.4 application to meet all of the following requirements (listed in no particular order):

* Use the latest version of Red Hat Universal Base Image 8 **ubi8** from the **registry registry.access.redhat.com** as a base.
* Install the **java-1.8.0-openjdk-devel** package.
* Create a system group for **jboss** with a GID of 1100.
* Create a system user for **jboss** with a UID 1100.
* Set the jboss user’s home directory to `/opt/jboss`.
* Set the working directory to jboss user’s home directory.
* Recursively change the ownership of the jboss user’s home directory to **jboss:jboss**.
* Expose port 8080.
* Make the container run as the **jboss** user.
* Unpack the **jboss-eap-7.4.0.zip** file to the `/opt/jboss` directory.
* Set the environment variable **JBOSS_HOME** to `/opt/jboss/jboss-eap-7.4`.
* Start container with the following executable: `/opt/jboss/jboss-eap-7.4/bin/standalone.sh -b 0.0.0.0 -c standalone-full-ha.xml`.

Before you start, log in to Red Hat Developer portal and download the **jboss-eap-7.4.0.zip** file to your working directory. See the weblink below (which requires a Red Hat account):

https://developers.redhat.com/content-gateway/file/jboss-eap-7.4.0.zip

### Solution to Podman Task 1

This is how the Dockerfile should look like. Comments are provided for references.

```
# Use base image
FROM registry.access.redhat.com/ubi8:latest

# Install the java-1.8.0-openjdk-devel package
# We also need the unzip package to unpack the JBoss .zip archive
RUN yum install -y java-1.8.0-openjdk-devel unzip && yum clean all

# Create a system user and group for jboss, they both have a UID and GID of 1100
# Set the jboss user's home directory to /opt/jboss
RUN groupadd -r -g 1100 jboss && useradd -u 1100 -r -m -g jboss -d /opt/jboss -s /sbin/nologin jboss

# Set the environment variable JBOSS_HOME to /opt/jboss/jboss-eap-7.4.0
ENV JBOSS_HOME="/opt/jboss/jboss-eap-7.4"

# Set the working directory to jboss' user home directory
WORKDIR /opt/jboss

# Unpack the jboss-eap-7.4.0.zip file to the /opt/jboss directory
ADD ./jboss-eap-7.4.0.zip /opt/jboss
RUN unzip /opt/jboss/jboss-eap-7.4.0.zip

# Recursively change the ownership of the jboss user's home directory to jbosis:jboss
# Make sure to RUN the chown after the ADD command and and before it, as ADD will
# create new files and directories with a UID and GID of 0 by default
RUN chown -R jboss:jboss /opt/jboss

# Make the container run as the jboss user
USER jboss

# Expose JBoss port
EXPOSE 8080

# Start JBoss, use the exec form which is the preferred form
ENTRYPOINT ["/opt/jboss/jboss-eap-7.4/bin/standalone.sh", "-b", "0.0.0.0", "-c", "standalone-full-ha.xml"]
```

### Podman Task 2: Build a JBoss Container Image and Run a Container

Use `podman` command to build a container image using the **Dockerfile** from task 1 and create a new container from the image to meet all of the following requirements:

* Tag the container image as **jboss-eap:7.4.0**.
* Create a new container in detached mode.
* Name the container **jboss-from-dockerfile**.
* Port forward from host port 38080 to container port 8080.
* Inspect the running container to determine its arguments (Args). You can use the following JSON format: `'{{.Args}}'`.
* Retrieve the last 15 lines of the running container logs.

### Solution to Podman Task 2

Make sure that the file **jboss-eap-7.4.0.zip** has been downloaded and is in the working directory:

```
$ ls -1
Dockerfile
jboss-eap-7.4.0.zip
```

Build the image:

```
$ podman build -t jboss-eap:7.4.0 .
```

View all local images:

```
$ podman images
REPOSITORY                       TAG         IMAGE ID      CREATED        SIZE
localhost/jboss-eap              7.4.0       04522ac82423  2 minutes ago  1.45 GB
registry.access.redhat.com/ubi8  latest      16fb1f57c5af  2 weeks ago    214 MB
```

Run a new container using the image:

```
$ podman run -d --name jboss-from-dockerfile -p 38080:8080 jboss-eap:7.4.0
```

Verify that the container is running:

```
$ podman ps
CONTAINER ID  IMAGE                      COMMAND     CREATED         STATUS             PORTS                    NAMES
a3e1f164097e  localhost/jboss-eap:7.4.0              37 seconds ago  Up 37 seconds ago  0.0.0.0:38080->8080/tcp  jboss-from-dockerfile
```

Inspect the running container:

```
$ podman inspect --format '{{.Args}}' jboss-from-dockerfile
[-b 0.0.0.0 -c standalone-full-ha.xml]
```

Retrieve the last 15 lines of the running container logs:

```
$ podman logs --tail=5 jboss-from-dockerfile
```

The task is complete.

### Podman Task 3: Save Container Image

Perform all of the following tasks:

* Save the **jboss-eap:7.4.0** container image to the **jboss-eap-7.4.0-backup.tar** file.
* Update the running **jboss-from-dockerfile** container and replace the content of the `/opt/jboss/jboss-eap-7.4/welcome-content/index.html` file with a single word JBOSS.
* Stop the running **jboss-from-dockerfile** container and commit your changes to create a new container image.
* The new container image should be called **jboss-eap** and have a tag of **7.4.0-dev**.
* Author name of the new container image should be set to "Home Lab".

### Solution to Podman Task 3

Save the image to a file:

```
$ podman save -o jboss-eap-7.4.0-backup.tar jboss-eap:7.4.0
Copying blob 33204bfe17ee done
Copying blob ca59617ea9ea done
Copying blob cf12abd501e1 done
Copying blob 8b490fc98797 done
Copying blob f3f0bffa6c8c done
Copying blob 8c9ee159685e done
Copying config 04522ac824 done
Writing manifest to image destination
Storing signatures
```

Verify that the backup file has been created:

```
$ ls -1
Dockerfile
jboss-eap-7.4.0-backup.tar
jboss-eap-7.4.0.zip
```

Connect to the running **jboss-from-dockerfile** container and update the index file as requested:

```
$ podman exec -it jboss-from-dockerfile /bin/bash
[jboss@a3e1f164097e ~]$ echo JBOSS > /opt/jboss/jboss-eap-7.4/welcome-content/index.html
[jboss@a3e1f164097e ~]$ cat /opt/jboss/jboss-eap-7.4/welcome-content/index.html
JBOSS
[jboss@a3e1f164097e ~]$ exit
```

Optionally, verify with `curl`:

```
$ curl http://127.0.0.1:38080
JBOSS
```

Stop the running **jboss-from-dockerfile** container image:

```
$ podman stop jboss-from-dockerfile
```

Commit the changes to a new **jboss-eap:7.4.0-dev** container image:

```
$ podman commit --author "Home Lab" jboss-from-dockerfile jboss-eap:7.4.0-dev
Getting image source signatures
Copying blob 33204bfe17ee skipped: already exists
Copying blob ca59617ea9ea skipped: already exists
Copying blob cf12abd501e1 skipped: already exists
Copying blob 8b490fc98797 skipped: already exists
Copying blob f3f0bffa6c8c skipped: already exists
Copying blob 8c9ee159685e skipped: already exists
Copying blob 0a00603d8194 done
Copying config ec66f8b31b done
Writing manifest to image destination
Storing signatures
ec66f8b31b6854285f7e5718e80df73d935da59d43afa7fa7eeda9adcbf5a1dc
```

Verify to make sure that the new image has been created:

```
$ podman images
REPOSITORY                       TAG         IMAGE ID      CREATED         SIZE
localhost/jboss-eap              7.4.0-dev   ec66f8b31b68  11 seconds ago  1.47 GB
localhost/jboss-eap              7.4.0       04522ac82423  32 minutes ago  1.45 GB
registry.access.redhat.com/ubi8  latest      16fb1f57c5af  2 weeks ago     214 MB
```

The task is complete.

### Podman Task 4: Push and Pull Images from Registries

For this task you are going to need to have a free account on Red Hat's **quay.io** repository so that you can push a container image to it. If you don't have one, feel free to use any other container image registry that you may have an account with (e.g. Docker Hub, AWS ECR etc).

Perform all of the following tasks:

* Use `podman` command to search and locate the **docker.io/httpd** official container image.
* Pull the latest version of the **httpd** container image from **docker.io** Docker Hub image repository.
* Use `podman` command to login to a Red Hat's **quay.io** image repository.
* Tag the **httpd** image as **quay.io/$USERNAME/httpd:1.0-test**, where $USERNAME is your **quay.io** username.
* Push the tagged container image to Red Hat's **quay.io** repository.

### Solution to Podman Task 4

Search for the official **httpd** image:

```
$ podman search --filter=is-official docker.io/httpd
NAME                     DESCRIPTION
docker.io/library/httpd  The Apache HTTP Server Project
```

Pull the latest version of the **httpd** image from Docker Hub:

```
$ podman pull docker.io/httpd:latest
Trying to pull docker.io/library/httpd:latest...
Getting image source signatures
Copying blob b1c114085b25 done
Copying blob 4691bd33efec done
Copying blob 9df1012343c7 done
Copying blob ff7b0b8c417a done
Copying blob a603fa5e3b41 done
Copying config 8653efc8c7 done
Writing manifest to image destination
Storing signatures
8653efc8c72daee8c359a37a1dded6270ecd1aede2066cbecd5be7f21c916770
```

Login to **quay.io** container image repository:

```
$ podman login quay.io
Username:
Password:
Login Succeeded!
```

Tag the **httpd:latest** image:

```
$ podman tag httpd:latest quay.io/lisenet/httpd:1.0-test
```

Push the image to **quay.io** image repository:

```
$ podman push quay.io/lisenet/httpd:1.0-test
Getting image source signatures
Copying blob 7603afd8f9aa done
Copying blob ec4a38999118 done
Copying blob a4a57da7ddfc done
Copying blob a0b242781abd done
Copying blob 29657939e55a done
Copying config 8653efc8c7 done
Writing manifest to image destination
Storing signatures
```

The task is complete.

### Podman Task 5: Deploy a WordPress Application

Use `podman` to deploy a new WordPress application with persistent database storage to meet all of the following requirements:

* You must run Podman containers as a non-root user (rootless).
* Create a new `$HOME/wpdb_podman` directory on the host.
* Apply the correct SELinux context type to the `$HOME/wpdb_podman` directory (and all subdirectories) to allow containers to access to all of its content.
* Create a new pod named **wp**.
* Create a new MySQL container named **wpdb** in detached mode.
* Use the tag **5.7-49** of the container image **registry.access.redhat.com/rhscl/mysql-57-rhel7**.
* The user that the MySQL container runs as is **UID=27** and **GID=27** (you can use `podman inspect` command to verify this).
* Set the following environment variables in the **wpdb** container:
    * MYSQL_ROOT_PASSWORD=ex180AdminPassword
    * MYSQL_USER=wpuser
    * MYSQL_PASSWORD=ex180UserPassword
    * MYSQL_DATABASE=wordpress
* Mount the host directory `$HOME/wpdb_podman` to the wpdb container path `/var/lib/mysql/data`.
* Create a new WordPress container named **wpapp** in detached mode.
* Use the latest version of the **docker.io/library/wordpress** container image.
* Set the following environment variables in the **wpapp** container:
    * WORDPRESS_DB_HOST=127.0.0.1
    * WORDPRESS_DB_USER=wpuser
    * WORDPRESS_DB_PASSWORD=ex180UserPassword
    * WORDPRESS_DB_NAME=wordpress
* Port forward from host port 8080 to the **wpapp** container port 80.
* Both **wpapp** and **wpdb** containers should belong to the **wp** pod.

### Solution to Podman Task 5

Create a directory to use for persistent storage:

```
$ mkdir -p $HOME/wpdb_podman
```

Check the current SELinux context of the directory:

```
$ ls -Zd $HOME/wpdb_podman
unconfined_u:object_r:user_home_t:s0 /home/user/wpdb_podman
```

Use `man` pages if you don't remember `semanage` syntax:

```
$ man semanage-fcontext
```

Set the new SELinux context:

```
$ sudo semanage fcontext -a -t container_file_t "$HOME/wpdb_podman(/.*)?"
$ sudo restorecon -Rv $HOME/wpdb_podman
```

Verify that the SELinux context has been applied:

```
$ ls -Zd $HOME/wpdb_podman
unconfined_u:object_r:container_file_t:s0 /home/user/wpdb_podman
```

Let us quickly verify the user of the MySQL container by inspecting the image:

```
$ podman inspect registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-49 | grep User
"User": "27",
"User": "27",
```

Because we have to use rootless containers, we need to change the UID/GID of the volume directory `$HOME/wpdb_podman` in the rootless Podman user namespace to match the UID/GID of the MySQL container user.

We can see below that the `$HOME/wpdb_podman` directory is owned by the UID/GID of 0, which is not the same UID/GID as the MySQL container **27**.

```
$ podman unshare ls -aln $HOME/wpdb_podman
total 4
drwxrwxr-x.  2 0 0    6 Nov 24 21:36 .
drwx------. 11 0 0 4096 Nov 24 21:33 ..
```

This means that the database container will fail to write to the directory. We need to fix it:

```
$ podman unshare chown -R 27:27 $HOME/wpdb_podman
```

Verify:

```
$ podman unshare ls -aln $HOME/wpdb_podman
total 4
drwxrwxr-x.  2 27 27    6 Nov 24 21:36 .
drwx------. 11  0  0 4096 Nov 24 21:33 ..
```

Create a new pod called **wp** with port forwarding and verify:

```
$ podman pod create --name wp -p 8080:80
```

```
$ podman pod ps
POD ID        NAME        STATUS      CREATED            INFRA ID      # OF CONTAINERS
44b9f3ce2b47  wp          Running     About an hour ago  6026e0eda6c0  1
```

Run a new MySQL container in an existing **wp** pod with persistent storage:

```
$ podman run -d \
  --pod wp \
  --name database \
  -e MYSQL_ROOT_PASSWORD="ex180AdminPassword" \
  -e MYSQL_USER="wpuser" \
  -e MYSQL_PASSWORD="ex180UserPassword" \
  -e MYSQL_DATABASE="wordpress" \
  --volume $HOME/wpdb_podman:/var/lib/mysql/data \
  registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-49
```

Make sure that the database container is running, and that the data directory has been populated with data:

```
$ podman logs --tail=1 database
Version: '5.7.24'  socket: '/var/lib/mysql/mysql.sock'  port: 3306  MySQL Community Server (GPL)
```

```
$ ls -ln $HOME/wpdb_podman/
total 41036
-rw-r-----. 1 100026 100026       56 Nov 24 21:42 auto.cnf
-rw-------. 1 100026 100026     1680 Nov 24 21:42 ca-key.pem
-rw-r--r--. 1 100026 100026     1112 Nov 24 21:42 ca.pem
-rw-r--r--. 1 100026 100026     1112 Nov 24 21:42 client-cert.pem
-rw-------. 1 100026 100026     1676 Nov 24 21:42 client-key.pem
-rw-r-----. 1 100026 100026      673 Nov 24 21:42 ib_buffer_pool
-rw-r-----. 1 100026 100026 12582912 Nov 24 21:42 ibdata1
-rw-r-----. 1 100026 100026  8388608 Nov 24 21:42 ib_logfile0
-rw-r-----. 1 100026 100026  8388608 Nov 24 21:42 ib_logfile1
-rw-r-----. 1 100026 100026 12582912 Nov 24 21:57 ibtmp1
drwxr-x---. 2 100026 100026     4096 Nov 24 21:42 mysql
-rw-r--r--. 1 100026 100026        6 Nov 24 21:42 mysql_upgrade_info
drwxr-x---. 2 100026 100026     8192 Nov 24 21:42 performance_schema
-rw-------. 1 100026 100026     1676 Nov 24 21:42 private_key.pem
-rw-r--r--. 1 100026 100026      452 Nov 24 21:42 public_key.pem
-rw-r--r--. 1 100026 100026     1112 Nov 24 21:42 server-cert.pem
-rw-------. 1 100026 100026     1680 Nov 24 21:42 server-key.pem
drwxr-x---. 2 100026 100026     8192 Nov 24 21:42 sys
drwxr-x---. 2 100026 100026       20 Nov 24 21:42 wordpress
-rw-r-----. 1 100026 100026        2 Nov 24 21:42 wp.pid
```

Run a new WordPress container in an existing **wp** pod:

```
$ podman run -d \
 --pod wp \
 --name application \
 -e WORDPRESS_DB_HOST="127.0.0.1" \
 -e WORDPRESS_DB_USER="wpuser" \
 -e WORDPRESS_DB_PASSWORD="ex180UserPassword" \
 -e WORDPRESS_DB_NAME="wordpress" \
  docker.io/library/wordpress:latest
```

Check to see if both containers are running:

```
$ podman ps
CONTAINER ID  IMAGE                                                   COMMAND               CREATED            STATUS                PORTS                 NAMES
6026e0eda6c0  localhost/podman-pause:4.1.1-1666713358                                       About an hour ago  Up About an hour ago  0.0.0.0:8080->80/tcp  44b9f3ce2b47-infra
c064b4fe26c9  registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-49  run-mysqld            29 minutes ago     Up 29 minutes ago     0.0.0.0:8080->80/tcp  database
947d4296255a  docker.io/library/wordpress:latest                      apache2-foregroun...  16 minutes ago     Up 16 minutes ago     0.0.0.0:8080->80/tcp  application
```

Check the number of containers in the pod:

```
$ podman pod ps
POD ID        NAME        STATUS      CREATED            INFRA ID      # OF CONTAINERS
44b9f3ce2b47  wp          Running     About an hour ago  6026e0eda6c0  3
```

Use `curl` to access the WordPress website:

```
$ curl -L http://127.0.0.1:8080
```

The task is complete.

## Creating Applications in OpenShift

We are going to use the EX180 exam objectives for OpenShift and perform 4 hands-on tasks to get familiar with an application deployment process.

### OpenShift Task 1: Deploy a MySQL Application using OpenShift Templates

* OpenShift can be accessed on the following URL: **https://api.crc.testing:6443**.
* Login to OpenShift cluster with username **developer** and password **developer**.
* Create a new project called **ex180task1** and use this project to deploy resources for this task.
* The OpenShift installer creates several templates by default. Use the **mysql-ephemeral** template to deploy a MySQL application to OpenShift.
* The name of the application should be **database**.
* Inspect the **mysql-ephemeral** template to work out what variables to pass during deployment.
* Set MySQL user to **dbuser1**.
* Set MySQL user password to **dbpass1**.
* Set MySQL database name to **db1**.
* Set MySQL administrator password **dbadminpass1**.
* Verify that the database **db1** has been created.

### Solution to OpenShift Task 1

Login to the OpenShift cluster and verify:

```
$ eval $(crc oc-env)
$ oc login -u developer -p developer https://api.crc.testing:6443
```

```
$ oc whoami
developer
```

Create a new project and verify:

```
$ oc new-project ex180task1
```

```
$ oc project
Using project "ex180task1" on server "https://api.crc.testing:6443"
```

As mentioned in the task definition, OpenShift creates several templates by default in the openshift namespace. List all available templates:

```
$ oc get templates -n openshift
NAME                                          DESCRIPTION                                                                        PARAMETERS        OBJECTS
cache-service                                 Red Hat Data Grid is an in-memory, distributed key/value store.                    8 (1 blank)       4
cakephp-mysql-example                         An example CakePHP application with a MySQL database. For more information ab...   21 (4 blank)      8
cakephp-mysql-persistent                      An example CakePHP application with a MySQL database. For more information ab...   22 (4 blank)      9
dancer-mysql-example                          An example Dancer application with a MySQL database. For more information abo...   18 (5 blank)      8
dancer-mysql-persistent                       An example Dancer application with a MySQL database. For more information abo...   19 (5 blank)      9
[...output truncated...]
mariadb-ephemeral                             MariaDB database service, without persistent storage. For more information ab...   8 (3 generated)   3
mariadb-persistent                            MariaDB database service, with persistent storage. For more information about...   9 (3 generated)   4
mysql-ephemeral                               MySQL database service, without persistent storage. For more information abou...   8 (3 generated)   3
mysql-persistent                              MySQL database service, with persistent storage. For more information about u...   9 (3 generated)   4
nginx-example                                 An example Nginx HTTP server and a reverse proxy (nginx) application that ser...   10 (3 blank)      5
[...output truncated...]
```

We see there is a template called **mysql-ephemeral** that we need to use to deploy a MySQL application.

There are two different ways that I'm aware of to list available parameters from a template. One is to describe a template using `oc describe` command like this:

```
$ oc describe template mysql-ephemeral -n openshift | less
```

Another one is to use `oc process` command like this:

```
$ oc process --parameters mysql-ephemeral -n openshift
NAME                    DESCRIPTION                                                             GENERATOR           VALUE
MEMORY_LIMIT            Maximum amount of memory the container can use.                                             512Mi
NAMESPACE               The OpenShift Namespace where the ImageStream resides.                                      openshift
DATABASE_SERVICE_NAME   The name of the OpenShift Service exposed for the database.                                 mysql
MYSQL_USER              Username for MySQL user that will be used for accessing the database.   expression          user[A-Z0-9]{3}
MYSQL_PASSWORD          Password for the MySQL connection user.                                 expression          [a-zA-Z0-9]{16}
MYSQL_ROOT_PASSWORD     Password for the MySQL root user.                                       expression          [a-zA-Z0-9]{16}
MYSQL_DATABASE          Name of the MySQL database accessed.                                                        sampledb
MYSQL_VERSION           Version of MySQL image to be used (8.0-el7, 8.0-el8, or latest).                            8.0-el8
```

We can have a look at the YAML template definition as well. This gives us some idea of what the DeploymentConfig would do to deploy the application.

```
$ oc get template mysql-ephemeral -n openshift -o yaml
```

Deploy MySQL application from the template:

```
$ oc -n ex180task1 new-app \
  --name=database \
  --template=mysql-ephemeral \
  --param=MYSQL_USER=dbuser1 \
  --param=MYSQL_PASSWORD=dbpass1 \
  --param=MYSQL_DATABASE=db1 \
  --param=MYSQL_ROOT_PASSWORD=dbadminpass1
```

Expected output:

```
--> Deploying template "openshift/mysql-ephemeral" to project ex180task1

     MySQL (Ephemeral)
     ---------
     MySQL database service, without persistent storage. For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/mysql-container/blob/master/8.0/root/usr/share/container-scripts/mysql/README.md.
     
     WARNING: Any data stored will be lost upon pod destruction. Only use this template for testing

     The following service(s) have been created in your project: mysql.
     
            Username: dbuser1
            Password: dbpass1
       Database Name: db1
      Connection URL: mysql://mysql:3306/
     
     For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/mysql-container/blob/master/8.0/root/usr/share/container-scripts/mysql/README.md.

     * With parameters:
        * Memory Limit=512Mi
        * Namespace=openshift
        * Database Service Name=mysql
        * MySQL Connection Username=dbuser1
        * MySQL Connection Password=dbpass1
        * MySQL root user Password=dbadminpass1
        * MySQL Database Name=db1
        * Version of MySQL Image=8.0-el8

--> Creating resources ...
    secret "mysql" created
    service "mysql" created
    deploymentconfig.apps.openshift.io "mysql" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/mysql' 
    Run 'oc status' to view your app.
```

Verify pod status:

```
$ oc get po
NAME             READY   STATUS      RESTARTS   AGE
mysql-1-deploy   0/1     Completed   0          110s
mysql-1-wwdcl    1/1     Running     0          107s
```

Verify that the **db1** database has been created. Connect to the MySQL pod, when inside the pod, login to MySQL to show databases:

```
$ oc exec -it po/mysql-1-wwdcl -- /bin/bash

bash-4.4$ mysql -u dbuser1 -pdbpass1 -e "show databases;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| db1                |
| information_schema |
+--------------------+
bash-4.4$ exit
```

The task is complete.

### OpenShift Task 2: Deploy a MySQL Application from a Template File

* OpenShift can be accessed on the following URL: **https://api.crc.testing:6443**.
* Login to OpenShift cluster with username **developer** and password **developer**.
* Create a new project called **ex180task2** and use this project to deploy resources for this task.
* Download OpenShift template from **https://raw.githubusercontent.com/openshift/origin/master/examples/db-templates/mysql-persistent-template.json** and save it to the **mysql-persistent-template.json** file.
* Create a new OpenShift template from the **mysql-persistent-template.json** file.
* Deploy a new MySQL application with persistent storage using the template file **mysql-persistent-template.json**.
* The name of the application should be **persistentdb**.
* Inspect the **mysql-persistent** template to work out what variables to pass during deployment.
* Set MySQL user to **dbuser2**.
* Set MySQL user password to **dbpass2**.
* Set MySQL database name to **db2**.
* Set MySQL administrator password **dbadminpass2**.
* Set volume space available for MySQL data to **1Gi**.
* Verify that a persistent volume claim (PVC) has been created.
* Copy the `/etc/my.cnf` file from the MySQL pod to the host directory `/tmp`.

### Solution to OpenShift Task 2

Login to the OpenShift cluster and verify:

```
$ eval $(crc oc-env)
$ oc login -u developer -p developer https://api.crc.testing:6443
```

```
$ oc whoami
developer
```

Create a new project and verify:

```
$ oc new-project ex180task2
```

```
$ oc project
Using project "ex180task2" on server "https://api.crc.testing:6443"
```

Download the template using `curl` and save it to the `mysql-persistent-template.json file`:

```
$ curl -sSf -o mysql-persistent-template.json \
  https://raw.githubusercontent.com/openshift/origin/master/examples/db-templates/mysql-persistent-template.json
```

Create a template from the `mysql-persistent-template.json` file:

```
$ oc -n ex180task2 create -f ./mysql-persistent-template.json
```

By default, the template will be created under the current project:

```
$ oc get templates
NAME               DESCRIPTION                                                                        PARAMETERS        OBJECTS
mysql-persistent   MySQL database service, with persistent storage. For more information about u...   9 (3 generated)   4
```

Get environment variables defined in the template:

```
$ oc process --parameters mysql-persistent
NAME                    DESCRIPTION                                                             GENERATOR           VALUE
MEMORY_LIMIT            Maximum amount of memory the container can use.                                             512Mi
NAMESPACE               The OpenShift Namespace where the ImageStream resides.                                      openshift
DATABASE_SERVICE_NAME   The name of the OpenShift Service exposed for the database.                                 mysql
MYSQL_USER              Username for MySQL user that will be used for accessing the database.   expression          user[A-Z0-9]{3}
MYSQL_PASSWORD          Password for the MySQL connection user.                                 expression          [a-zA-Z0-9]{16}
MYSQL_ROOT_PASSWORD     Password for the MySQL root user.                                       expression          [a-zA-Z0-9]{16}
MYSQL_DATABASE          Name of the MySQL database accessed.                                                        sampledb
VOLUME_CAPACITY         Volume space available for data, e.g. 512Mi, 2Gi.                                           1Gi
MYSQL_VERSION           Version of MySQL image to be used (8.0-el7, 8.0-el8, or latest).                            8.0-el8
```

We see that the variable used to define volume space is called VOLUME_CAPACITY.

Deploy MySQL application from the template file `mysql-persistent-template.json`:

```
$ oc -n ex180task2 new-app \
  --name=persistentdb \
  --file=mysql-persistent-template.json \
  --param=MYSQL_USER=dbuser2 \
  --param=MYSQL_PASSWORD=dbpass2 \
  --param=MYSQL_DATABASE=db2 \
  --param=MYSQL_ROOT_PASSWORD=dbadminpass2 \
  --param=VOLUME_CAPACITY=1Gi
```

Expected output:

```
--> Deploying template "ex180task2/mysql-persistent" for "mysql-persistent-template.json" to project ex180task2

     MySQL
     ---------
     MySQL database service, with persistent storage. For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/mysql-container/blob/master/8.0/root/usr/share/container-scripts/mysql/README.md.
     
     NOTE: Scaling to more than one replica is not supported. You must have persistent volumes available in your cluster to use this template.

     The following service(s) have been created in your project: mysql.
     
            Username: dbuser2
            Password: dbpass2
       Database Name: db2
      Connection URL: mysql://mysql:3306/
     
     For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/mysql-container/blob/master/8.0/root/usr/share/container-scripts/mysql/README.md.

     * With parameters:
        * Memory Limit=512Mi
        * Namespace=openshift
        * Database Service Name=mysql
        * MySQL Connection Username=dbuser2
        * MySQL Connection Password=dbpass2
        * MySQL root user Password=dbadminpass2
        * MySQL Database Name=db2
        * Volume Capacity=1Gi
        * Version of MySQL Image=8.0-el8

--> Creating resources ...
    secret "mysql" created
    service "mysql" created
    persistentvolumeclaim "mysql" created
    deploymentconfig.apps.openshift.io "mysql" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/mysql' 
    Run 'oc status' to view your app.
```

Verify pod status:

```
$ oc get po
NAME             READY   STATUS      RESTARTS   AGE
mysql-1-deploy   0/1     Completed   0          2m25s
mysql-1-zdks7    1/1     Running     0          2m22s
```

List persistent volume claims:

```
$ oc get pvc
NAME    STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql   Bound    pv0004   100Gi      RWO,ROX,RWX                   2m50s
```

Copy `/etc/my.cnf` from a remote pod to `/tmp` locally:

```
$ oc cp mysql-1-zdks7:/etc/my.cnf /tmp/my.cnf
```

This task is complete.

### OpenShift Task 3: Deploy a PHP Application using Source-to-Image (S2I)

* OpenShift can be accessed on the following URL: **https://api.crc.testing:6443**.
* Login to OpenShift cluster with username **developer** and password **developer**.
* Create a new project called **ex180task3** and use this project to deploy resources for this task.
* Create a new PHP Hello World application using S2I deployment.
* The name of the application should be **helloworld**.
* The source code is available from the Git repository at **https://github.com/lisenet/DO180-apps/** in the **php-helloworld** directory.
* Use the **development** branch of the Git repository.
* Use the **php:7.3** image stream tag.
* Expose the PHP application’s service to make the application accessible from a web browser.

### Solution to OpenShift Task 3

Login to the OpenShift cluster and verify:

```
$ eval $(crc oc-env)
$ oc login -u developer -p developer https://api.crc.testing:6443
```

```
$ oc whoami
developer
```

Create a new project and verify:

```
$ oc new-project ex180task3
```

```
$ oc project
Using project "ex180task3" on server "https://api.crc.testing:6443"
```

Deploy a new PHP application using S2I:

```
$ oc -n ex180task3 new-app \
  --name helloworld \
  -i php:7.3 https://github.com/lisenet/DO180-apps#development \
  --context-dir php-helloworld
```

Expected output:

```
--> Found image 70d4d94 (2 months old) in image stream "openshift/php" under tag "7.3" for "php:7.3"

    Apache 2.4 with PHP 7.3 
    ----------------------- 
    PHP 7.3 available as container is a base platform for building and running various PHP 7.3 applications and frameworks. PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developers to write dynamically generated web pages. PHP also offers built-in database integration for several commercial and non-commercial database management systems, so writing a database-enabled webpage with PHP is fairly simple. The most common use of PHP coding is probably as a replacement for CGI scripts.

    Tags: builder, php, php73, rh-php73

    * The source repository appears to match: php
    * A source build using source code from https://github.com/lisenet/DO180-apps#development will be created
      * The resulting image will be pushed to image stream tag "helloworld:latest"
      * Use 'oc start-build' to trigger a new build

--> Creating resources ...
    imagestream.image.openshift.io "helloworld" created
    buildconfig.build.openshift.io "helloworld" created
Warning: would violate PodSecurity "restricted:v1.24": allowPrivilegeEscalation != false (container "helloworld" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "helloworld" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "helloworld" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "helloworld" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
    deployment.apps "helloworld" created
    service "helloworld" created
--> Success
    Build scheduled, use 'oc logs -f buildconfig/helloworld' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/helloworld' 
    Run 'oc status' to view your app.
```

Check the build config:

```
$ oc get bc
NAME         TYPE     FROM              LATEST
helloworld   Source   Git@development   1
```

Check the service:

```
$ oc get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
helloworld   ClusterIP   10.217.4.118   none          8080/TCP,8443/TCP   37s
```

Check the image stream:

```
$ oc get is
NAME         IMAGE REPOSITORY                                                                TAGS     UPDATED
helloworld   default-route-openshift-image-registry.apps-crc.testing/ex180task3/helloworld   latest   About a minute ago
```

Check the pods:

```
$ oc get po
NAME                          READY   STATUS      RESTARTS   AGE
helloworld-1-build            0/1     Completed   0          3m3s
helloworld-66879f45bc-gfc7h   1/1     Running     0          109s
```

Note: you can use `oc get all` command to list all resources.

Expose the PHP application:

$ oc expose svc helloworld

Make sure that the route resource has been created:

```
$ oc get routes
NAME         HOST/PORT                                PATH   SERVICES     PORT       TERMINATION   WILDCARD
helloworld   helloworld-ex180task3.apps-crc.testing          helloworld   8080-tcp                 None
```

Access the application with curl:

```
$ curl http://helloworld-ex180task3.apps-crc.testing
Hello, World! php version is 7.3.33
This is development branch
```

This task is complete.

### OpenShift Task 4: Deploy an HTTP Application from Image

* OpenShift can be accessed on the following URL: **https://api.crc.testing:6443**.
* Login to OpenShift cluster with username **developer** and password **developer**.
* Create a new project called **app-from-image** and use this project to deploy resources for this task.
* Deploy a new web application using the image you have built for the [Podman task 4](#podman-task-4-push-and-pull-images-from-registries). Use private image **quay.io/$USERNAME/httpd:1.0-test**, where $USERNAME is your quay.io username.
* Create an OpenShift secret called **my-quay-credentials** for use with a container registry, and use your quay.io credentials. Use this secret to deploy the application from the private registry.
* The name of the application should be **webserver**.
* Add the following label to all resources: **task=4**.
* Create the application as a **deployment config**.
* Troubleshoot and identify the reason why the pod is failing to start.
* Delete the project from OpenShift.

### Solution to OpenShift Task 4

Login to the OpenShift cluster and verify:

```
$ eval $(crc oc-env)
$ oc login -u developer -p developer https://api.crc.testing:6443
```

```
$ oc whoami
developer
```

Create a new project and verify:

```
$ oc new-project app-from-image
```

```
$ oc project
Using project "app-from-image" on server "https://api.crc.testing:6443"
```

Check documentation for how to create a new secret:

```
$ oc create secret --help
```

Create a new secret to store quay.io credentials:

```
$ oc -n app-from-image create secret docker-registry my-quay-credentials \
  --docker-server=quay.io \
  --docker-username=YOUR_QUAY_USERNAME \
  --docker-password=YOUR_QUAY_PASSWORD \
  --docker-email=YOUR_QUAY_EMAIL
```

Verify that the secret has been created:

```
$ oc get secret my-quay-credentials -o yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJxdWF5LmlvIjp7InVzZXJuYW1lIjoiWU9VUl9RVUFZX1VTRVJOQU1FIiwicGFzc3dvcmQiOiJZT1VSX1FVQVlfUEFTU1dPUkQiLCJlbWFpbCI6IllPVVJfUVVBWV9FTUFJTCIsImF1dGgiOiJXVTlWVWw5UlZVRlpYMVZUUlZKT1FVMUZPbGxQVlZKZlVWVkJXVjlRUVZOVFYwOVNSQT09In19fQ==
kind: Secret
metadata:
  creationTimestamp: "2022-11-27T18:50:40Z"
  name: my-quay-credentials
  namespace: app-from-image
  resourceVersion: "161791"
  uid: 918119ae-1f6c-455d-8002-73bcb4bfbef3
type: kubernetes.io/dockerconfigjson
```

Optionally, we could decode the data by using `base64 -d` if we wanted to see the credentials.

To use the secret for pulling images for pods, we must add the secret to the default service account, because the service account that the pod uses is the default service account.

Link secrets to a service account:

```
$ oc secrets link default my-quay-credentials --for=pull
```

Next we can check documentation for how to add labels to resource and use a deployment config:

```
$ oc new-app --help | less
```

Create the webserver application using the httpd image from a private registry and specify the existing secret to use:

```
$ oc -n app-from-image new-app \
  --name webserver \
  --labels=task=4 \
  --as-deployment-config=true \
  --image=quay.io/lisenet/httpd:1.0-test
```

Expected output:

```
--> Found container image 8653efc (12 days old) from quay.io for "quay.io/lisenet/httpd:1.0-test"

    * An image stream tag will be created as "webserver:1.0-test" that will track this image
    * This image will be deployed in deployment config "webserver"
    * Port 80/tcp will be load balanced by service "webserver"
      * Other containers can access this service through the hostname "webserver"
    * WARNING: Image "quay.io/lisenet/httpd:1.0-test" runs as the 'root' user which may not be permitted by your cluster administrator

--> Creating resources with label task=4 ...
    imagestream.image.openshift.io "webserver" created
    deploymentconfig.apps.openshift.io "webserver" created
    service "webserver" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/webserver' 
    Run 'oc status' to view your app.
```

Get deployment config:

```
$ oc get dc
NAME        REVISION   DESIRED   CURRENT   TRIGGERED BY
webserver   1          1         1         config,image(webserver:1.0-test)
```

Get pod status including labels:

```
$ oc get po
NAME                 READY   STATUS             RESTARTS        AGE   LABELS
webserver-1-deploy   0/1     Completed          0               17m   openshift.io/deployer-pod-for.name=webserver-1
webserver-1-kc22v    0/1     CrashLoopBackOff   7 (4m26s ago)   17m   deployment=webserver-1,deploymentconfig=webserver,task=4
```

We see that the webserver pod status is CrashLoopBackOff.

Get the cluster events to see if anything is failing:

```
$ oc get events
```

Get the webserver pod logs:

```
$ oc logs po/webserver-1-kc22v
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.217.0.173. Set the 'ServerName' directive globally to suppress this message
(13)Permission denied: AH00072: make_sock: could not bind to address [::]:80
(13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:80
no listening sockets available, shutting down
AH00015: Unable to open logs
```

It looks like the container is trying to bind to a privileged port 80, but is not allowed to do so. The task did not ask us to resolve the problem, but identify the cause.

Delete the project:

```
$ oc delete project app-from-image
```

The task is complete.
