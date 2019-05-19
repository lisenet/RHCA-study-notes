######################################################################
## Tomas Nevar <tomas@lisenet.com>
## Study notes for EX403 Satellite 6 Deployment and Systems Management
######################################################################

## Exam objectives:

* Install Red Hat Satellite Server from an ISO image
* Create and configure organizations and locations
* Create and configure Red Hat Satellite users and roles
* Create GNU Privacy Guard (GPG) keys to sign RPMs
* Create and configure development life cycles
* Create a Puppet product repository
* Configure Red Hat Satellite subscriptions, content, and content views
* Install a Red Hat Satellite capsule server using an ISO image
* Build a custom RPM from a source tarball
* Create a Red Hat Satellite activation key
* Create a Red Hat Satellite host group
* Define smart class parameters for a Puppet module
* Register an existing client to a Red Hat Satellite server
* Apply errata to client
* Configure bare-metal deployment on Red Hat Satellite
* Deploy clients using kick starts

## Red Hat Satellite 6 is a federation of several upstream open source
## projects, including Foreman, Katello, Pulp and Candlepin.

* Foreman: provisioning on new clients.
* Pulp: patch and content (package repository) management.
* Candlepin: subscription and entitlement management.
* Katello: unified workflow and WebUI for Pulp and Candlepin.
* Puppet: configuration management.
* Hammer: CLI tool providing shell equivalents of most Web UI functions.

## Red Hat Satellite 6 hardware and software requirements:

* The latest version of RHEL 6 server or 7 server.
* A minimum of 2 CPU cores, but 4 CPU cores are recommended.
* A minimum of 8 GB memory but ideally 12 GB of memory.
* Use 4 GB of swap space where possible.
* No Java virtual machine installed on the system.
* No Puppet RPM files installed on the system.
* No third-party unsupported yum repositories enabled.
* Full forward and reverse DNS resolution using a FQDN.
* At least 10GB of free disk space for disconnected installations.

## A lot of the exam objectives are covered in my homelab:

https://www.lisenet.com/2018/homelab-project-with-kvm-katello-and-puppet/

#---------------------------------------------------------------------

## Install Red Hat Satellite Server from an ISO image.

# firewall-cmd --permanent --add-service={RH-Satellite-6,dhcp,dns,tftp}
# firewall-cmd --permanent --add-port=68/udp
# firewall-cmd --permanent --add-port=8000/tcp
# firewall-cmd --reload

# wget http://example.com/satellite-6-dvd.iso
# mount -o loop ./satellite-6-dvd.iso /mnt

## Package installation may take up to 10 minutes:

# cd /mnt && ./install_packages
# cd && umount /mnt

## It may take around 20 minutes to install the server. Use
## satellite-installer to enable/configure DNS, DHCP and TFTP services:

# satellite-installer \
  --scenario satellite \
  --foreman-admin-username admin \
  --foreman-admin-password password \
  --foreman-proxy-dns true \
  --foreman-proxy-dhcp true \
  --foreman-proxy-tftp true \
  --foreman-proxy-dns-interface eth0 \
  --foreman-proxy-dhcp-interface eth0 \
  --foreman-proxy-dns-zone ex403.hl.local \
  --foreman-proxy-dns-forwarders 10.11.1.2 \
  --foreman-proxy-dns-reverse 1.11.10.in-addr.arpa \
  --foreman-proxy-dhcp-range "10.11.1.90 10.11.1.95" \
  --foreman-proxy-dhcp-gateway 10.11.1.1 \
  --foreman-proxy-dhcp-nameservers 10.11.1.70

## On the Satellite server, generate a certificate for the capsule: 

# capsule-certs-generate --capsule-fqdn capsule.hl.local \
  --certs-tar ~/capsule.hl.local.tar

 To finish the capsule installation, follow these steps:

  satellite-installer --scenario capsule\
    --capsule-parent-fqdn                  "satellite.hl.local"\
    --foreman-proxy-register-in-foreman    "true"\
    --foreman-proxy-foreman-base-url       "https://satellite.hl.local"\
    --foreman-proxy-trusted-hosts          "satellite.hl.local"\
    --foreman-proxy-trusted-hosts          "capsule.hl.local"\
    --foreman-proxy-oauth-consumer-key     "A1EVNBrjxaW9qKV7omorF6nU43BjcMM0"\
    --foreman-proxy-oauth-consumer-secret  "B2tj8K26M8yLTTCFtQrWFAyxb28ssHc0"\
    --capsule-pulp-oauth-secret            "C3nPpEru7UuwMNGDnjRkzGrVNwTGwA90"\
    --capsule-certs-tar                    "/root/capsule.hl.local.tar"
  The full log is at /var/log/capsule-certs-generate.log

# scp /root/capsule.hl.local.tar capsule.hl.local:~/

#---------------------------------------------------------------------

## Install a Red Hat Satellite capsule server using an ISO image.

# firewall-cmd --permanent --add-service={RH-Satellite-6,dhcp,dns,tftp}
# firewall-cmd --permanent --add-port=68/udp
# firewall-cmd --permanent --add-port=8000/tcp
# firewall-cmd --reload

# wget http://example.com/satellite-capsule-6-dvd.iso
# mount -o loop ./satellite-capsule-6-dvd.iso /mnt
# cp /mnt/media.repo /etc/yum.repos.d/capsule.repo
# echo "baseurl=file:///mnt/" >> /etc/yum.repos.d/capsule.repo
# chmod 0644 /etc/yum.repos.d/capsule.repo
# yum install -y satellite-capsule

# Register capsule to Satellite under the Default Organization:

# yum -y localinstall \
  http://satellite.hl.local/pub/katello-ca-consumer-latest.noarch.rpm
# subscription-manager register --org=Default_Organization

## Configure capsule as a content node using the satellite-installer
## command that was displayed previously in the output of the
## capsule-certs-generate command executed on the Satellite server:

# satellite-installer --scenario capsule \
 --capsule-parent-fqdn                  "satellite.hl.local" \
 --foreman-proxy-register-in-foreman    "true" \
 --foreman-proxy-foreman-base-url       "https://satellite.hl.local" \
 --foreman-proxy-trusted-hosts          "satellite.hl.local" \
 --foreman-proxy-trusted-hosts          "capsule.hl.local" \
 --foreman-proxy-oauth-consumer-key     "A1EVNBrjxaW9qKV7omorF6nU43BjcMM0" \
 --foreman-proxy-oauth-consumer-secret  "B2tj8K26M8yLTTCFtQrWFAyxb28ssHc0" \
 --capsule-pulp-oauth-secret            "C3nPpEru7UuwMNGDnjRkzGrVNwTGwA90" \
 --capsule-certs-tar                    "/root/capsule.hl.local.tar"

## It may take around 5 minutes to configure the capsule.

#---------------------------------------------------------------------

## Register an existing client to a Red Hat Satellite server.

# yum -y localinstall \
  http://satellite.hl.local/pub/katello-ca-consumer-latest.noarch.rpm
# subscription-manager clean
# subscription-manager register --org=Default_Organization
# yum -y install katello-agent

## Once a client has been registered to Satellite server, the web UI
## can be used to modify the client's configuration.

#---------------------------------------------------------------------

## Build a custom RPM from a source tarball.

## The source code must be in the form of a compressed archive before
## starting the RPM build process!

## In order to package an RPM, you need to:

1. Download the source code.
2. Create the spec file (use the rpmdev-newspec command).
3. Build the package (use the rpmbuild command).
4. GPG sign the package (use the rpmsign command).
5. Test the package (use rpm -qip command).

## RPM packaging process:

$ sudo yum install rpmdevtools rpm-build rpm-sign rpmlint gcc rng-tools

$ wget http://katello.hl.local/homelab-1.0.tar.gz
$ tar xf homelab-1.0.tar.gz
$ less homelab/README
$ less homelab/Makefile

$ rpmdev-setuptree
$ cp homelab-1.0.tar.gz rpmbuild/SOURCES/
$ cd rpmbuild/SPECS
$ rpmdev-newspec homelab
$ vim homelab.spec
$ rpmlint homelab.spec
$ rpmbuild -ba homelab.spec

#---------------------------------------------------------------------

## Create GNU Privacy Guard (GPG) keys to sign RPMs.

$ sudo rngd -r /dev/urandom
$ gpg --gen-key
$ gpg --fingerprint

$ echo '%_gpg_name <your_gpg_fingerprint>' >> ~/.rpmmacros
$ rpmsign --addsign ~/rpmbuild/RPMS/x86_64/homelab-1.0-1.el7.x86_64.rpm
$ rpm -qip ~/rpmbuild/RPMS/x86_64/homelab-1.0-1.el7.x86_64.rpm

$ sudo yum localinstall -y --nogpgcheck \
  ~/rpmbuild/RPMS/x86_64/homelab-1.0-1.el7.x86_64.rpm

#---------------------------------------------------------------------

## Create and configure organizations and locations.

# hammer organization create --name Lisenet
# hammer location create --name HomeLab

#---------------------------------------------------------------------

## Create and configure Red Hat Satellite users and roles.

# hammer user create \
  --login alice \
  --firstname Alice \
  --lastname Abernathy \
  --mail "alice@hl.local" \
  --organizations Lisenet \
  --locations HomeLab \
  --roles Manager \
  --password changeMe \
  --auth-source-id 1

#---------------------------------------------------------------------

## Create and configure development life cycles.

# hammer lifecycle-environment create \
  --organization Lisenet \
  --name dev \
  --prior Library

# hammer lifecycle-environment create \
  --organization Lisenet \
  --name qa \
  --prior dev

# hammer lifecycle-environment create \
  --organization Lisenet \
  --name prod \
  --prior qa

#---------------------------------------------------------------------

## Create a Puppet product repository.

# hammer product create \
  --organization Lisenet \
  --name puppet_stuff \
  --description "Puppet modules"

# hammer repository create \
  --organization Lisenet \
  --name puppet_repo \
  --content-type puppet \
  --product puppet_stuff

# hammer repository upload-content \
  --organization Lisenet \
  --name puppet_repo \
  --product puppet_stuff \
  --path /root/lisenet-lisenet_firewall-1.0.0.tar.gz

#---------------------------------------------------------------------

## Configure Red Hat Satellite subscriptions, content and content views.

# hammer content-view create \
  --name puppet_modules \
  --description "Content view for Puppet modules"

# hammer content-view puppet-module add \
  --content-view puppet_modules \
  --name lisenet_firewall

# hammer content-view publish \
  --name puppet_modules \
  --description "Publishing Puppet modules"

# hammer content-view version promote \
  --content-view puppet_modules \
  --version 1.0 \
  --to-lifecycle-environment dev

#---------------------------------------------------------------------

## Create a Red Hat Satellite activation key.

# hammer activation-key create \
  --name el7-key \
  --description "Key to use with EL7" \
  --lifecycle-environment dev \
  --content-view puppet_modules \
  --unlimited-hosts

# hammer activation-key add-subscription \
  --name el7-key \
  --quantity 1 \
  --subscription-id 1

#---------------------------------------------------------------------

## Create a Red Hat Satellite host group.

# hammer hostgroup create \
  --query-organization Lisenet \
  --locations HomeLab \
  --name el7_group \
  --description "Host group for EL7 servers" \
  --lifecycle-environment dev

# hammer hostgroup set-parameter  \
  --name kt_activation_keys \
  --value el7-key \
  --hostgroup el7_group

#---------------------------------------------------------------------

## Define smart class parameters for a Puppet module.

https://www.lisenet.com/2018/katello-working-with-puppet-modules-and-creating-the-main-manifest/

#---------------------------------------------------------------------

## Apply errata to client.

Satellite WebUI > Content > Errata
Pick requried errata > Apply Errata > Apply to Content Hosts > Next

#---------------------------------------------------------------------

## Configure bare-metal deployment on Red Hat Satellite.

## We may need to implement a workaround so that the PXE configuration
## for booting off of the local disk would not be misinterpreted by the
## new host. In order to configure boot from first hard drive, we need
## to remove the LOCALBOOT 0 entry and replace it with the following:

  COM32 chain.c32
  APPEND hd0

## chain.c32 is a COM32 module for Syslinux. It can chainload MBRs,
## partition boot sectors, Windows bootloaders etc.

## If the workaround is not applied, the server will still be provisioned,
## but might fail to boot off of the local disk.

#---------------------------------------------------------------------

## Deploy clients using kick starts.

# hammer template list
# hammer template kinds

# hammer template dump \
  --id "Katello Kickstart Default" > custom_template.txt

# vim custom_template.txt

# hammer template create \
  --organizations Lisenet \
  --locations HomeLab \
  --file custom_template.txt \
  --name "Katello Kickstart Default Custom" \
  --type provision \
  --operatingsystems "CentOS 7.4.1708"

# hammer host create \
  --name proxy1 \
  --hostgroup el7_group \
  --interface "type=interface,mac=00:22:ff:00:00:19,ip=10.11.1.19,managed=true,primary=true,provision=true"

#---------------------------------------------------------------------

## Timeframes for creating various things in the lab:

# 10 min to install Satellite packages from DVD.
# 15-20 min to install the Satellite server.
# 5 min to install and configure the capsule server.
# 20-40 min to synchronise RPM repositories (depending on their size).
# 10-20 min for RHEL 7 DVD sync from repo discovery.
# 5 min to publish content views.
# 3-5 min to promote content views to lifecycle environments.
# 10-15 min to kickstart provision a RHEL 7 server using PXE boot.

## This should help manage expectations during the exam.
