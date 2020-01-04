# Study notes for EX405 Configuration Management with Puppet (RHEL7)
_by Tomas Nevar (tomas@lisenet.com)_

## Exam objectives:

* Install and configure Puppet.
  - Install Puppet servers.
  - Install Puppet nodes.
  - Register Puppet nodes to a Puppet server.
* Create and maintain Puppet manifests.
  - Create new Puppet manifests.
  - Debug existing Puppet manifests.
* Create Puppet modules.
  - Create reusable modules.
  - Create modules with classes, name spaces, variables, and conditionals.
  - Create modules that install software on target nodes and deploy configuration files.
* Use facter to obtain system information.
  - Create custom facts.
  - Use facts to change Puppet behavior.
* Create Git repositories.
  - Create and perform simple management of a Git repository.
  - Add files to a Git repository.
  - Apply changes and commit changed files to a Git repository.
* Implement Puppet in a Red Hat Satellite 6 environment.
  - Create a Puppet repository on Red Hat Satellite.
  - Install, configure, and deploy Puppet modules using Red Hat Satellite.
  - Register Puppet clients to a Red Hat Satellite server.

## 1. Puppet Agent
A Puppet run starts with the Puppet agent gathering facts on the Puppet node. The facts are sent over HTTPS to the Puppet master. 

The Puppet master compiles a catalog and sends the catalog back to the Puppet node. The Puppet node applies the catalog and sends a report to the Puppet master. 

Install a Puppet agent:
```
$ sudo yum -y install puppet
```
Show available subcommands (output truncated):
```
$ puppet help
> agent         The puppet agent daemon
> apply         Apply Puppet manifests locally
> cert          Manage certificates and requests
> describe      Display help about resource types
> doc           Generate Puppet references
> facts         Retrieve and store facts
> generate      Generates Puppet code from Ruby definitions
> man           Display Puppet manual pages
> module        Creates, installs and searches for modules on the Forge
> node          View and manage node definitions
> parser        Interact directly with the parser
> resource      The resource abstraction layer shell
```

## 2. Puppet Resources
Puppet resource titles can contain any characters, but they are case-sensitive.

List available resource types:
```
$ puppet resource --types
> exec
> file
> group
> notify
> package
> service
> user
```
Listing Puppet resource types only provides the names of resources. To see the details, use the `puppet describe` command:
```
$ puppet describe exec
$ puppet describe service
```

Getting information about a specific resource type:
```
$ puppet resource <type> <name>
```
```
$ puppet resource service sshd
service { 'sshd':
  ensure => 'running',
  enable => 'true',
}
```
Note that file resources create files owned by root with default permissions of `0644`. Directories have default permissions of `0755` when they are created. We can define different default values for a resource:
```
File {
  ensure => 'file',
  owner  => 'tomcat',
  group  => 'tomcat',
  mode   => '600',
}
```
If an attribute block ends with a semicolon `;` rather than a comma `,` then another title, colon and attribute block can be specified. Puppet will treat this as multiple resources of a single type.

To apply a standalone Puppet manifest to the local system:
```
$ puppet apply manifest.pp
```
Do a dry-run (a basic smoke test). This will ensure that a manifest can be properly compiled from your code:
```
$ puppet apply --noop manifest.pp
```
IMPORTANT! Validate the syntax of a manifest:
```
$ puppet parser validate manifest.pp
```
To save time, create an alias and add to `~/.bashrc`:
```
alias ppv='puppet parser validate'
```

## 3. Puppet Modules
Modules are self-contained bundles of code and data. You can write your own modules, or you can download pre-built modules from Forge. Module names should only contain lowercase letters, numbers, and underscores, and should begin with a lowercase letter.

A module is simply a directory tree with a specific structure:
```
<MODULE NAME>
  manifests
  files
  templates
  lib
  facts.d
  tests
  spec
```
Each manifest in a module's manifests folder should contain one class or defined type. Also, every module includes a JSON-formatted `metadata.json` information file.

Show available subcommands:
```
$ puppet module --help
> build        Build a module release package
> changes      Show modified files of an installed module
> generate     Generate boilerplate for a new module
> install      Install a module from the Forge or a release archive
> list         List installed modules
> search       Search the Puppet Forge for a module
> uninstall    Uninstall a puppet module
> upgrade      Upgrade a puppet module
```

Create a template directory structure for writing a module:
```
$ puppet module generate <author-modulename>
```

Build a Puppet module. The module will be built and packaged in the pkg directory in the module's working directory:
```
$ puppet module build <author-modulename>
```
Install a module from a release archive (requires root privileges):
```
# puppet module install <author-modulename>/pkg/<author-modulename>.tgz
```

## 4. Puppet Classes
Classes are named blocks of Puppet code, which are stored in modules for later use and are not applied until they are invoked by name. The `include` command is used in a manifest to use a class.

Example of a simple class definition:
```
class webserver {
  package { 'httpd':
    ensure  => 'present',
  }
}
```
Example of a simple class with parameters:
```
class webserver ($version = 'latest') {
  package {'httpd':
    ensure => $version, # Using the class parameter from above
  }
}
```
Note that every Puppet module should have a smoke test defined in `tests/init.pp` that includes the main class defined by the module.

About relationships and ordering. By default, Puppet applies resources in the order they're declared in their manifest. Puppet uses four metaparameters to establish relationships:

* before - applies a resource before the target resource.
* require - applies a resource after the target resource.
* notify - applies a resource before the target resource. The target resource refreshes if the notifying resource changes.
* subscribe - applies a resource after the target resource. The subscribing resource refreshes if the target resource changes.

The service will be started after the package has been installed:
```
service { 'vsftpd':
  ensure => 'running',
  enable => true,
  require => Package['vsftpd'],
}
package { 'vsftpd':
  ensure => 'installed',
}
```
You can create relationships between two resources or groups of resources using the `->` and `~>` operators.
```
-> (ordering arrow; a hyphen and a greater-than sign) - explained below
~> (notifying arrow; a tilde and a greater-than sign) - explained below


-> Applies the resource on the left before the resource on the right
~> Applies the resource on the left first. If the left-hand resource
   changes, the right-hand resource will refresh.
```
The service will be refreshed if the config file changes:
```
package { 'vsftpd':
  ensure => present,
} ->
file { '/etc/vsftpd/vsftpd.conf':
  ensure => file,
  mode   => '0600',
  source => 'puppet:///modules/vsftpd/vsftpd.conf',
} ~>
service { 'vsftpd':
  ensure => running,
  enable => true,
}
```

## 5. Puppet Templates
Puppet supports two templating languages: EPP and ERB. ERB works with all Puppet versions.

You can put template files in the templates directory of a module. EPP files should have the .epp extension, and ERB files should have the `.erb` extension. Here is example below:
```
file {'/eth/hosts':
  ensure => 'file',
  owner => 'root',
  group => 'root',
  mode => '644',
  # loads /etc/puppet/modules/homelab/templates/hosts.erb
  content => template('homelab/hosts.erb'),
}
```
Example content of the `hosts.erb` template file:
```
127.0.0.1   localhost 
::1         localhost
<%= @ipaddress %>   <%= @fqdn %>
```

## 6. Regular Expressions
Puppet uses Ruby's standard regular expression implementation to match patterns. Regular expressions are written as patterns bordered by forward slashes.

* `==` operator checks for exact string matches.
* `=~` operator is used to evaluate whether a string matches regex.
* `!~` operator performs the opposite (does not match regex).

Examples:

```
if $operatingsystemmajrelease == '7' {
  notify { "This is a RHEL 7 server": }
}
```
```
if $fqdn =~ /^web[1-3].hl.local$/ {
  notify { "This is a web server": }
}
```

## 7. Facter
The first thing the Puppet agent does when it begins a Puppet run is to identify system facts about the node it is running on.

Display all facts about the system:
```
$ facter
```
Display a single structured fact:
```
$ facter os
```
Display a single fact nested within a structured fact:
```
$ facter os.family
$ facter os.release.major
```
Some commonly used facts (from my experience):
```
$ facter architecture
$ facter fqdn
$ facter id
$ facter ipaddess
$ facter osfamily
$ facter operatingsystem
$ facter operatingsystemmajrelease
$ facter is_virtual
```

Facter has the capability to provide additional custom facts. Additional system key=value pairs can be defined either in a text file, YAML or JSON files in `/etc/facter/facts.d`.

Note that custom scripts can also provide Facter with dynamic facts that are collected on the fly. These scripts must be executable!

Create a custom system fact:
```
# mkdir -p /etc/facter/facts.d
# echo "server_group=tomcat" > /etc/facter/facts.d/server_group.txt
```
```
$ facter | grep server_group
server_group => tomcat
```

## 8. Puppet Master
Install a Puppet master:
```
# yum install -y puppet-server
# systemctl enable puppetmaster
# systemctl start puppetmaster
# firewall-cmd --permanent --add-port=8140/tcp
# firewall-cmd --reload
```
Configure Puppet agent to use the Puppet master:
```
# vim /etc/puppet/puppet.conf
[agent]
  server = puppet-master.example.com
  runinterval = 1800
```
```
# systemctl enable puppet
# systemctl start puppet
```
The Puppet agent will generate a host certificate and send a certificate-signing request to the Puppet master. We need to sign the client certificate on the Puppet master:
```
# puppet cert list
# puppet cert sign puppet-agent.example.com
```

## 9. Satellite Server
Register Puppet clients to a Red Hat Satellite server. These are copied from my homelab Katello server:
```
# wget http://katello.hl.local/pub/katello-ca-consumer-latest.noarch.rpm
# yum install -y katello-ca-consumer-latest.noarch.rpm
# subscription-manager register --org=lisenet --activationkey="el7-key"
# yum install -y katello-agent
```
Create a Puppet repository on Red Hat Satellite. These are copied from my homelab Katello server:
```
# hammer product create \
  --name "puppet_stuff" \
  --description "Puppet modules"
```
```
# hammer repository create \
  --name "puppet_repo" \
  --content-type "puppet" \
  --product "puppet_stuff"
```
```
# hammer repository upload-content \
  --name "puppet_repo" \
  --product "puppet_stuff" \
  --organization "Lisenet" \
  --path /root/lisenet-lisenet_firewall-1.0.0.tar.gz
```
```
# hammer content-view create \
  --name "puppet_modules" \
  --description "Content view for Puppet modules"
```
```
# hammer content-view puppet-module add \
  --content-view "puppet_modules" \
  --name "lisenet_firewall"
```
```
# hammer content-view publish \
  --name "puppet_modules" \
  --description "Publishing Puppet modules"
```
```
# hammer content-view version promote \
  --content-view "puppet_modules" \
  --version "1.0" \
  --to-lifecycle-environment "stable"
```
```
# hammer activation-key create \
  --name "el7-key" \
  --description "Key to use with EL7" \
  --lifecycle-environment "stable" \
  --content-view "puppet_modules" \
  --unlimited-hosts
```
```
# hammer activation-key add-subscription \
  --name "el7-key" \
  --quantity "1" \
  --subscription-id "1"
```

## 10. Git
Show available git subcommands (output truncated):
```
$ git --help
> add        Add file contents to the index
> branch     List, create, or delete branches
> checkout   Checkout a branch or paths to the working tree
> clone      Clone a repository into a new directory
> commit     Record changes to the repository
> diff       Show changes between commits, commit and working tree, etc
> init       Create an empty Git repository or reinitialize an existing one
> log        Show commit logs
> mv         Move or rename a file, a directory, or a symlink
> pull       Fetch from and merge with another repository or a local branch
> push       Update remote refs along with associated objects
> reset      Reset current HEAD to the specified state
> rm         Remove files from the working tree and from the index
> show       Show various types of objects
> status     Show the working tree status
```

Set the name you want attached to your commit transactions:
```
$ git config --global user.name <username>
```
Set the email you want attached to your commit transactions:
```
$ git config --global user.email <email>
```
Set the default push action if no refspec is explicitly given:
```
$ git config --global push.default simple
```
List all configured global values:
```
$ git config --global --list
```
Create a new local repository:
```
$ git init <repository-name>
```
Clone an existing repository:
```
$ git clone <repository-url>
```
Add all current changes to the next commit:
```
$ git add ./
```
Show all new or modified files to be commited:
```
$ git status
```
Show all commits:
```
$ git log
```

Show all existing branches:
```
$ git branch -av
```
Switch to the specified branch and update the working directory:
```
$ git checkout <branch-name>
```
Delete the specified branch:
```
$ git branch -d <branch-name>
```
Show all currently configured remotes:
```
$ git remote -v
```
Prepare file for the next commit:
```
$ git add <file>
```
Commit a specific file adding a message:
```
$ git commit -m "My commit message" <file>
```
Upload changes made locally to the remote repository:
```
$ git push <remote> <branch>
```
Delete the file from the working directory and stage the deletion:
```
$ git rm <file>
```
Remove the file from the staging area (the opposite of git add):
```
$ git reset <file>
```
Revert the file to its original state:
```
$ git checkout -- <file>
```
Shows content differences:
```
$ git diff 
```
Fetch changes made by other users from the repository:
```
$ git pull
``
