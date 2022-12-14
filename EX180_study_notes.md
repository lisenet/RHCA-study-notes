# Study notes for EX180 Red Hat Certified Specialist in Containers and Kubernetes
_by Tomas Nevar (tomas@lisenet.com)_

## Exam objectives:

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