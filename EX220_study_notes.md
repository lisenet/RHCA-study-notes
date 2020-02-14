# Study notes for EX220 Hybrid Cloud Management
_by Tomas Nevar (tomas@lisenet.com)_

## Exam objectives:

* Perform initial configuration of the CloudForms Appliance.
  - Configure CloudForms database and network settings.
  - Perform initial CloudForms configuration and configure server roles.
* Connect to one or more resource providers.
  - Connect to a Red Hat Virtualization infrastructure provider.
  - Connect to a Red Hat OpenStack Platform cloud provider.
* Configure authentication and security.
  - Create and utilize tenants.
  - Create and utilize users, groups, and roles.
* Selectively grant access to virtual systems.
* Provision virtual systems.
  - Provision virtual machines.
  - Create provisioning key pairs.
* Create volumes and assign them to virtual machines.
* Use cloud-init to customize virtual machines.
* Perform other management tasks.
  - Create and apply SmartTags.
  - Configure chargeback reporting.
  - Configure reporting.
  - Perform drift analyses.
  - Configure alerts.

## 1. Perform Initial Configuration of the CloudForms Appliance

[Product Documentation for Red Hat CloudForms 4.7](https://access.redhat.com/documentation/en-us/red_hat_cloudforms/4.7/) ([General Configuration](https://access.redhat.com/documentation/en-us/red_hat_cloudforms/4.7/html-single/general_configuration/index), [Appliance Hardening Guide](https://access.redhat.com/documentation/en-us/red_hat_cloudforms/4.7/html-single/appliance_hardening_guide/index))

### Configure CloudForms Database and Network Settings

Basic configuration of the appliance is done via SSH.

```
$ ssh root@cfme.lab.example.com
```

Launch the configuration manager:

```
# appliance_console
```

Follow the on-screen instructions.

Note that it is advisable to change the default **root** password. Continuing to use the default password leaves the appliance vulnerable to any user attempting to gain root access.

### Perform Initial CloudForms Configuration and Configure Server Roles

Use the web interface.

```
Settings → Configuration
```

Red Hat CloudForms uses a unique **admin** user to control all functions in the web-based user interface. After installing the appliance, change the default password of the **admin** to restrict administrative access to the appliance's UI. 

```
Settings → Configuration → Access Control → Users → Administrator
Configuration → Edit this User → Change Password
```

## 2. Connect to One or More Resource Providers

Red Hat CloudForms 4 supports multiple providers.

### Add an Infrastructure Provider

```
Compute → Infrastructure → Providers → Configuration → Add a New Infrastructure Provider 
```

Select **Red Hat Enterprise Virtualization Manager** as the Type.

Add host creadentials:
```
Compute → Infrastructure → Hosts
```
Configure SmartProxy Affinity to allow the Red Hat CloudForms appliance to perform SmartState analysis.
```
Settings → Configuration → then selectZone: Default Zone (current) → SmartProxy Affinity
```
Check **Server: EVM [1000000000001]** which will check all resources managed by it.

### Add a Cloud and Network Provider
```
Compute → Clouds → Providers → Configuration → Add a New Cloud Provider 
```
Select **OpenStack** as the provider. Click the Events tab to review the two available event endpoints CloudForms can use to gather metrics from the environment. Make sure Ceilometer is selected.

### Add a Containers Provider
```
Compute → Containers → Providers → Configuration → Add a New Containers Provider 
```
Select **OpenShift Enterprise** as the Type. You will need to generate an authentication token for CloudForms. Also configure the metrics collection setting by clicking the Hawkular tab.

## 3. Configure Authentication and Security

### Create and Utilize Tenants

Since CloudForms 4.0, all user operations are performed within the context of a tenant. The default top-level tenant is known as tenant zero, and all of the out-of-the-box groups are members of this tenant.

Tenants can have projects as children.

```
Settings → Configuration → Access Control → Tenants
```

### Create and Utilize Users, Groups and Roles

Users must belong to a group. Each group has a role, a tenant and filters assigned to it.

A role is applied to a group and defines what users can do with each product feature that is visible on the web interface.

```
Settings → Configuration → Access Control → Roles → Configuration menu → Add a new Role
```
```
Settings → Configuration → Access Control → Groups → Configuration menu → Add a new Group
```
```
Settings → Configuration → Access Control → Users → Configuration menu → Add a new User
```

## 4. Selectively Grant Access to Virtual Systems
List all VMs:
```
Compute → Infrastructure → Virtual Machines
```
Select the VM that you want to grant access to, and then set the owner to some user:
```
Configuration → Set Ownership
```

## 5. Provision Virtual Systems

### Provision Virtual Machines
```
Compute → Infrastructure → Virtual Machines → Lifecycle → Provision VMs
```
Provisioning VMs using a cloud provider:
```
Compute → Clouds → Instances → Lifecycle → Provision Instances
```

### Create Provisioning Key Pairs
```
Compute → Clouds → Key Pairs → Configuration → Add a New Key Pair
```

## 6. Create Volumes and Assign them to Virtual Machines

```
Compute → Clouds → Volumes → Configuration → Add a New Cloud Volume
```

## 7. Use Cloud-Init to Customize Virtual Machines
Cloud-Init can be used when provisioning virtual machines that have been deployed using a template. Note tha Cloud-Init files use YAML syntax.
```
Compute → Infrastructure → PXE → Customization Templates → Configuration → Add a New Customization Template
```

## 8. Perform Other Management Tasks

### Create and Apply SmartTags

SmartTags are used during automation, compliance testing and provisioning. Note: tags may be case sensitive.

```
Settings → Configuration → Settings accordion → CFME Region: Region 1 [1] → Red Hat Training Categories tab
```
Click anywhere in the table's first row to create a new category.

```
Settings → Configuration → Settings accordion → CFME Region: Region 1 [1] → Red Hat Training Tags tab
```
Create tags in the category.

### Configure Chargeback Reporting

Chargeback rates can either be set for Compute or Storage. Set chargeback rates:
```
Cloud Intel → Chargeback → Rates
```
Assign the new chargeback rates to virtual machines that belong to the default tenant.
```
Cloud Intel → Chargeback → Assignments
```

### Configure Reporting
```
Cloud Intel → Reports → All Reports → Configuration → Add a new Report 
```
### Perform Drift Analyses
To perform a SmartState analysis, CloudForms requires a running SmartProxy with visibility to the storage location of the virtual machine.

If at least two sets of SmartState analysis data exist for a virtual machine, those data points can be compared to spot changes. This is referred as drift reporting.

### Configure Alerts

Some predefined alerts are available with CloudForms.
```
Control → Explorer → Alerts → Add a New Alert
```

## 9. Misc

### Check Service Requests
```
Service → Requests
```
### RHV

The Red Hat Virtualization (RHV) host needs to be active to start the course.
