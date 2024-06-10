# Training Task 5. Ansible:
## Usage

The only dependency for this script that is not included in the default ansible pip3 install is the geerlingguy.nodejs role which can be downloaded from Ansible Galaxy using:

> `ansible-galaxy role install geerlingguy.nodejs`

Ansible connects to remote hosts via ssh connection, it therefore requires hosts to be active on the network before it can run any tasks on those hosts. This means there is some manual configuration needed before we can run our playbook. First we need to set up all of our VMs so that their NICs are active on the network. This can be done with the builtin linux network manager by running the command:

> `nmcli con mod ens33 ipv4.method manual ip4 10.1.1.2/24 ipv4.gateway 10.1.1.1`

This example modifies the default connection on the ens33 network interface, giving it a static IP address of 10.1.1.2 (this is the IP we have chosen for our dhcp server) and telling it that the IP address of our gateway server is 10.1.1.1. This needs to be done on our dns and app server too, assigning them IP addresses 10.1.1.3 and 10.1.1.4 respectively.

Since our gateway VM will have an automatically configured connection from the local EAS network dhcp, We do not need to manually configure its external network adapter to run tasks on it. The internal NIC does need to be configured but this is handled by the Ansible playbook.

We now need to copy our ssh public key to each of the VMs on our network so that ansible can carry connect to each host without a password. If we add a line to our inventory file to tell ansible to use an ssh jump to the hosts via the gateway then we only need to copy the public key from our dev machine to each of the hosts:

> `[gethinnet:vars] `<br>` 
> ansible_ssh_common_args='-J root@172.16.6.51'`

We can copy the public key with an ssh jump to each of the hosts using the ssh-copy-id command with some extra arguments:

> `ssh-copy-id -o ProxyJump=root@172.16.6.51 root@10.1.1.4`

Here, -o is used to give configuration options, namely the ProxyJump option which specifies the IP address and user through which to make our connection to the remote host, in this case our gateway is specified as the proxy for a jump to our app server.

The final thing to do is to update our yum mirrors. Since CentOS 8 packages were removed from the original official mirrors, we need to manually specify new mirrors that yum will search for its packages to install. We can do this with the linux sed commands:

> `sed -i -e ’s/mirrorlist=/#mirrorlist=/g’ /etc/yum.repos.d/CentOS-*`

And:

> `sed -i -e ’s|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*`

This should be run on each of our host VMs. But is required on the dhcp and dns servers for the playbook to function.

We have used Ansible vault to store our GitHub personal access token, this means we need to provide a password for the encrypted vault file so that the access token can be used by Ansible in the playbook. We have created a password file on the dev machine which we have been passing to Ansible when we run the playbook with the command:

> `ansible-playbook playbook.yml --vault-password-file ~/.ansible/github_token_pass.txt`

However, without this password file, the playbook could also be run with the command:

> `ansible-playbook playbook.yml --ask-vault-pass`

And by inputting the password for the vault file when prompted.

## Discussion:
### Objective:
Develop an ansible playbook for an In-House VMware Estate, the playbook will automate the deployment of a basic application within the VM subnet used in previous tasks. The playbook will be designed to run on the CentOS 8 stream template in vCenter and will be executable from our development MAC Book, with all the nessecary access and permissions to perform the deployment. The playbook will be built for scalability and best practices for security will be implemented.

### About Ansible:
Ansible is a tool that simplifies complex deployment tasks by automating the deployment of a network’s architecture. Ansible allows us to define our infrastructure as code, speeding up the building process and ensuring consistency and reliability.

### How Ansible Works:
Ansible temporarily connects to a remote host via ssh to do its tasks. A user specifies playbooks (containing configuration instructions) and an inventory (specifying target hosts and groups). The playbooks are executed by the control node on the target host.

### Installing Ansible:
The ansible docs are really useful for an installation guide:
https://docs.ansible.com/ansible/2.9/installation_guide/intro_installation.html

The first step to using ansible is to install ansible. We need to install ansible on our dev machine (referred to in ansible documentation as a control node.) Ansible then uses ssh to communicate with the devices we want to automate (managed nodes).

Ansible requires Python 3 to run so we need to install it on our dev machine. We can do this with brew:

> `brew install python3`
 
We also have SELinux installed by default on our VM template so we may need to install 
libselinux-python on our managed nodes so that we can carry out certain functions in ansible.

The preferred way to install Ansible on a Mac is with the python package manager pip. Before we can install and run ansible we need to ensure that we are working within a python virtual environment. We can do this by running the following command to create a new virtual environment and activating it in the current shell session:

> `alias pythonVenv="python3 -m venv .venv --prompt A && source .venv/bin/activate`

we can then install Ansible with the command:

> `pip3 install ansible`

### Creating an Inventory:
Ansible uses an inventory file to keep track of its managed nodes. We can create a new inventory on the dev machine and add a file using touch:

> `touch ~/Documents/Training/Task5/inventory`

Inventory files work by creating a group into which servers can be added. If we spin up a new vm which will be our gateway and get its IP we can add it to the inventory file with vim:

> `[gethinnet]`<br>
> `172.16.6.51`

Here we have defined the group ‘gethinnet’ in the square brackets and specified a server within that group at the IP 172.16.6.51 (our gateway IP on the EAS network).

We can now tell ansible to ping our server to make sure our inventory is working with the command:

> `ansible -i inventory gethinnet -m ping -u root`

To ping the server using ansible instead of ssh, we should get the following response:

> `172.16.6.51 | SUCCESS => {`<br>
> `  "ansible_facts": {`<br>
> `    "discovered_interpreter_python": "/usr/libexec/platform-python"`<br>
> `  },`<br>
> `  "changed": false,`<br>
> `  "ping": "pong"`<br>
> `}`

In our command we have had to specify the name of our inventory file, the group within the inventory file and the user on the VM to run the command as. We can set the user in our inventory file so that ansible will always run commands as the root user on our VM:

>`[gethinnet]`<br>
>`172.16.6.51 ansible_user=root`

And we can create an ansible.cfg file in the same location as our inventory file to give ansible some general configuration directives:

>`[defaults]`<br>
>`INVENTORY = inventory`

We can also assign aliases to our hosts to avoid the need to specify IP addresses. We create an inventory entry for our alias and give it a variable ansible_host to assign an IP to the host.

>`[easnet]`<br>
>`gateway ansible_host=root@172.16.6.51`

>`[gethinnet]`<br>
>`dhcp ansible_host=root@10.1.1.2`<br>
>`dns ansible_host=root@10.1.1.3`<br>
>`appa ansible_host=root@10.1.1.4`<br>

We can also assign variables to multiple groups at the same time by specifying a parent group ‘multi’ as follows:

>`[multi:children]`<br>
>`easnet`
>`gethinnet`
>
>`[multi:vars]`
>`ansible_user=root`

### YAML Syntax:
Ansible scripts are written in YAML syntax. YAML files optionally start with “---” and end with “…” these indicate the start and end of a document.

YAML files for ansible are structured using lists of key/value pairs called a dictionary. All members of a list are lines beginning at the same indentation level starting with a dash and a space “- ”. A dictionary is represented in “key: value” form. Ansible playbooks often consists of lists of dictionaries, dictionaries whose values are lists or a mix of both. 

An example of a list in yaml is shown below:

>`Item_a`<br>
>`Item_b`<br>
>`Item_c`<br>
>`Item_d`<br>

While a dictionary looks like this:

>`key_a: “value a”`<br>
>`key_b: “value b”`<br>
>`key_c: “value c”`<br>
>`key_d: “value d”`<br>

Boolean values can be represented by true/false, 1/0, or yes/no.

Values can span multiple lines with a literal block scalar “|” which includes newlines and trailing spaces in the lines following it, or a folding block scalar “>” will fold newlines to spaces, making what would be a very long line easier to read and edit. Indentation will be ignored in both cases.

Basic variables in yaml are called ‘scalars’ scalars generally do not need to be surrounded by quotes but there are some exceptions. A colon followed by a space or newline “: ” is an indicator for a mapping, while a space followed by a pound sign starts a comment. Scalars that include such characters need to be surrounded by quotes. If the scalar is enclosed in double quotes then escapes can be used.

Ansible uses “{{ var }}” to refer to variables. Double quotes are needed so that ansible doesn’t think that the brackets after a colon begin a dictionary. If a value in a dictionary starts with quotes then the whole value must be enclosed in quotes.

Other special characters that cannot be used in an unquoted scalar are:

>`[] {} > | * & ! % # `` @ , ? : -`

Yaml will also convert certain strings containing decimal points (e.g. 1.0) into floating point variables. If we want these to remain as strings we need to enclose them in quotes.

### Playbooks:
Ansible commands are run in YAML scripts called playbooks, these contain information on what machines on the network to run operations on and what to run on them.

A playbook in Ansible is made up of elements called ‘plays’ a play generally consists of three elements. The first element is a name which acts as a label for the play within the script and is displayed when the playbook is run to aid in debugging.

>`- name: Reconfigure dns and app network adapters to use dhcp settings`

The second element is the hosts key which allows the developer to specify which nodes from in the inventory file the following operations should be carried out on in the form of a list:

>`hosts:`
>	`- dns`
>	`- appa`

The second key is tasks, which contains a list of all the operations that Ansible should carry out on the specified host machine:

>  `tasks:`<br>
>  `  - name: Set NICs to use dhcp`<br>
>  `    community.general.nmcli:`<br>
>  `      state: present`<br>
>  `      conn_name: ens33`<br>
>  `      method4: auto`<br>
>   
>  `  - name: Restart network managers`<br>
>  `    ansible.builtin.service:`<br>
>  `      name: NetworkManager`<br>
>  `      state: restarted`<br>


Each task should consist of a name, and a module to run, followed by an indented dictionary of all of the parameters to pass to the module. 

### Ansible Modules:
All of the code that Ansible executes is divided into modules, each module has a particular use and can be invoked with a particular task or several can be invoked in a playbook. When Ansible is installed with pip, a number of the most commonly used modules come preinstalled which do not need to be separately installed, including the ansible.builtin collection which contains all of the core ansible functionality, the ansible.posix collection which contains functionality for a variety of common UNIX command packages and the community.general collection which contains community maintained modules that are not part of more specialised community collections.

### Ansible Variables:
Ansible uses variables to allow the execution of tasks on multiple different systems with a single command.

A variable name can only include letters, numbers and underscores and cannot begin with a number. A variable can be defined using standard YAML syntax:

>`my_variable: blah`

Once defined a variable can be referenced using Jinja2 syntax which uses double curly braces:

>`my variable is {{ my_variable }}`

As covered in the YAML syntax section, The curly brackets must be enclosed in quotations to be interpreted correctly by YAML.

A key:value dictionary variable can be referenced using individual, specific fields from that dictionary using either bracket notation or dot notation:

>`foo:[‘field1’]`<br>
>`Foo.field1`

When ansible connects to a remote host it carries out a process it calls ‘gather_facts.’ This process collates a list of facts about the remote host, its hardware and its configuration into a nested variable called ansible_facts. Values in a nested variable need to be referenced slightly differently to simple variables and bracket or dot notation must be used:

>`{{ ansible_facts[“ens33”][“ipv4”][“address”] }}`

Variables can be defined in a playbook, in an inventory file, in reusable files, in roles and at the command line.

For our Ansible playbook we have decided to include our variables in a separate file which we have passed to ansible in our playbook using the vars_files: command:

>`   vars_files:`<br>
>`    - vars/main.yml`<br>
>`    - vars/github_access_token.yml`<br>

Where the contents of our variables files are simple YAML dictionaries:

>`nodejs_package_json_path: "/root/Data"`<br>
>`app_port: "8000"`<br>
>`url: "https://jsonplaceholder.typicode.com/posts"`<br>

### Remote Environment:
The ‘environment’ keyword can be used at the play, block, or task level to set an environment variable for an action on a remote host, the variable will then be available to tasks within the play or block that are executed by the same user:

>`  environment:`<br>
>`    PORT: "{{ app_port }}"`<br>
>`    TARGET_URL: "{{ url }}"`<br>

### Ansible Vault:
Ansible Vault allows developers to keep sensitive data (like the GitHub personal access token used in our script) in encrypted files rather than as plaintext in playbooks or roles.

Ansible vault is controlled with the command line tool “ansible-vault” and various command line flags to control functionality.

We can create a variable file containing a simple YAML dictionary to store our sensitive variables before encrypting the file using:

>`ansible-vault encrypt vars/github_access_token.yml`

The encrypted file can then be included in our “vars_files” keyword and variables within it can be referenced like normal variables:

>`    - name: Download app files from github`<br>
>`      ansible.builtin.git:`<br>
>`        repo: "https://{{ TOKEN }}@github.com/Enterprise-Automation/trainee-challenge-node-app.git"`<br>
>`        dest: /root/Data`

If we want to then run our playbook we need to create a plain text password file containing only our password string and passing this file to ansible vault when we run our ansible-playbook command:

>`ansible-playbook playbook.yml --vault-password-file ~/.ansible/github_token_pass.txt`

### Ansible Roles:
Ansible’s roles feature allows users to group related tasks together in a way that allows them to be easily reused and shared.

An Ansible role has a defined directory structure with seven main standard directories, at least one of these directories must be included in a role and any of the directories that the role does not use can be omitted.

A role will look in its directories for a main.yml file by default. It will look in:

* “tasks/main.yml” for a list of tasks that the role provides to the play for execution
* “Handlers/main.yml” for handlers that are imported into the parent play for use by the role or other roles and tasks in the play
* “defaults/main.yml” for low precedence variables provided by the role that can be overwritten by any other variable source
* “vars/main.yml” for high precedence variables provided by the role to the play.
* “files/stuff.txt” for files that are made available to the role and its children.
* “templates/something.j2” for templates to use in the role or child roles.
* “meta/main.yml” for role metadata including dependencies and ansible galaxy data that is required for uploading a role to ansible galaxy but not to use the role in our play.

Any other files must be included explicitly using include_role/import_role.

By default ansible looks for roles in a directory called roles/ relative to the playbook file. A roles_path variable can be configured which ansible can also search.

Roles can be used in a few ways:

* At the play level with the roles option
* At the tasks level with the include_role option to dynamically use the role
* At the tasks level using import_role to use the role statically

Roles can be reused anywhere in the tasks section of a play using include_role. Roles added in the roles: section of a playbook run before any other task while included roles run in the order they are defined.

Ansible pre-processes all static imports during playbook parsing time while dynamic includes are processed during runtime at the point at which the task is encountered so if we need our role to react to varying data such as in a loop then we would include our role in the loop rather than importing it.

### Ansible Galaxy:
Ansible galaxy is a free site for finding, downloading and reviewing community-developed ansible roles. The ansible-galaxy client is included in Ansible and can be used to download roles from Ansible Galaxy and create a default framework for creating roles locally.

### Idempotence:
An important attribute of ansible is that it is idempotent, this means that if a change has already been made, and the playbook is run again without anything being different, then the change will not be made again. We have built our playbook to try to ensure that our tasks are idempotent however certain modules like ansible.builtin.git or ansible.builtin.command are not idempotent by nature and will always report a change.
