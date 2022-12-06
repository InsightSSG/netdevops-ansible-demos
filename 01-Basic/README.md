# Basic Ansible NetDevOps

This lab assumes that you are using the Cisco DevNet Sandbox titled Cisco Modeling Labs (CML): Enterprise, 
although you can build this lab in your own environment using GNS3, CML, or Eve-NG.

This lab has two major sections: 
   
**Ad-hoc commands** - In this section we will look at using ansible directly from the CLI to execute
 commands using the RAW module (e.g. using SSH directly).  Multiple examples will be given to include showing commands, 
 parsing commands using grep, and outputting commands to text files.
 
**Simple playbooks** - This section will expand on the previous section by creating reusable playbooks which can be stored 
and shared for cross-team use.  Much more advanced functionality is allowed in a playbook but it comes with additional 
complexity. We will look at three Cisco modules in this section: cisco.ios.ios_commands, cisco.ios.ios_config, and 
cisco.ios.ios_facts.

NOTE: These steps are not intended to be used in a production network. They are only offered as a guide for 
practicing in a lab environment.  Consult with an ansible professional for using ansible in production environments.


## Configuration and Inventory File

Change into the 01-Basic directory.
```
cd 01-Basic
```
Ansible will use the .cfg file located in this directory it was running from if present otherwise it will default to the 
file located in the /etc directory.  

For our simple example we only want to make sure that host_key_checking = False.  

The inventory file however must be specified. It is stored as a YAML file with the 
.yml extension.

Look at the contents of the simple inventory that is included.  This inventory file will reference all of the IOS devices
in the sandbox lab.  Type the cat command to get the output
```
cat inventory.yml
```

Output of **inventory.yml**
```
# Individual devices can be specified with variables added directly inline
10.10.20.175 ansible_network_os=cisco.ios.ios

# Create group called devnet_ios and add all IOS-XE devices to it
[devnet_ios]
dist-rtr01 ansible_host=10.10.20.175
dist-rtr02 ansible_host=10.10.20.176

# Assign variables to the group called devnet_ios.  ansible_network is well known.  psn1/2/3 are custom variables
[devnet_ios:vars]
ansible_network_os=cisco.ios.ios

# Create second group called devnet_nxos and assign NXOS devices to it
[devnet_nxos]
dist-sw01 ansible_host=10.10.20.177
dist-sw02 ansible_host=10.10.20.178

# Assign unique variables to devnet_nxos group
[devnet_nxos:vars]
ansible_network_os=cisco.nxos.nxos

# Create parent group called devnet that includes the child groups devnet_ios and devnet_nxos
[devnet:children]
devnet_ios
devnet_nxos

# Define variables applicable to all devices in inventory file.  All that start with ansible_ are well known.
#  psn_encrypted is a custom variable custom.
[all:vars]
ansible_become=yes
ansible_become_method=enable
ansible_user=cisco
ansible_password=cisco
ansible_connection=ansible.netcommon.network_cli
```

Notice that each of the devices are located under a grouping for the device type.  .175 and .176 which are both IOS-XE devices 
are specfied as **devnet_ios** whereas .177 and .178 are specified as part of the **devnet_nxos** group.  Variables are either 
defined in line with the device or can be associated with groups or the built in group all.  Multiple groups can be grouped 
together using the [<group_name>:children] notation.  

## host_vars and group_vars

Not all variables used in these labs will be stored in the inventory file.  Variables can be stored in either group specific 
files in the group_vars directory bearing a name that matches the associated group (e.g. group devnet should have file 
group_vars/devnet.yml).  Ansible will automatically search the host_vars and group_vars directory wherever the playbook was 
run from.  host_vars should be specified as 

## Ad-hoc commands
Ansible can be used directly from the CLI that it has been installed using the `ansible` command.  For full details on 
command line options see the documentation of run ```ansible --help```.  

### Run basic show commands using the RAW module
Ansible RAW module (ansible.builtin.raw) is a ansible-core module included in all Ansible installations.  Per the 
documentation it *Executes a low-down and dirty SSH command* on a device.  The primary use case is for devices that don't 
have Python installed which should include almost all network equipment (routers, switches, firewalls, load balancers, etc).  

We will be using the [Ansible RAW Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/raw_module.html) 
in our first few examples from the CLI.   

#### Basic show commands
From the CLI of the DevBox run the following command.  The flags are as follows:

|Flag|Description|
|:---|:---|
|-m| Module used (using raw)|
|-a | Argument used |
| -u | Username |
| -k | prompt for password| 

Command structure for basic show commands
```
ansible <<hostname>> -m raw -a "<<command>>" -u <<username>> -k
```

1. Run a basic show command against a single device
    ```
    ansible 10.10.20.175 -i inventory_basic.yml -m raw -a "show run" -u cisco -k
    ``` 

2. Target group with show command in inventory file
    - Change the group **all** to group **devnet_ios**.
    ```
    ansible devnet_ios -i inventory_basic.yml -m raw -a "show run" -u cisco -k
    ```

3. Change show command with piping
    ```
    ansible devnet_ios -i inventory_basic.yml -m raw -a "show version | inc Version|uptime|License, " -u cisco -k
    ``` 
   
4. Extend ad-hoc commands with **grep**
   - Search for file version running by looking for specific output from 'show version' to capture for both IOS-XE and 
   NX-OS devices.  
   - Specify multiple search terms using ```\|``` to separate them.
   - NOTE: Added ```SUCCESS\|CHANGED\|``` to grep statement so we can see the line which shows the login status and hostname.
   ```
    ansible all -i inventory_basic.yml -m raw -a "show version" -u cisco -k | grep 'SUCCESS\|CHANGED\|XE Software,\|NXOS image'
    ```
   
5. Extend ad-hoc commands with **grep**
   - Search for usernames in running by looking for specific output from 'show run' to capture for both IOS-XE and 
   NX-OS devices.  
   ```
    ansible all -i inventory_basic.yml -m raw -a "show run" -u cisco -k | grep 'SUCCESS\|CHANGED\|username'
    ```
   
6. Other non-privileged commands
   - Ping the same IP from multiple devices
   ```
    ansible all -i inventory_basic.yml -m raw -a "ping 172.16.252.21" -u cisco -k
    ```
## Simple Playbooks - Beginning with cisco.ios.commands

Next step is to interact with Cisco devices much like we have, but using playbooks.  A playbook is a repeatable, re-usable 
config management/deployment file.  With [playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html) 
we can execute tasks, request command output, manipulate commands and more.  Ansible can use many modules from multiple 
vendors in playbooks but we will be using the [cisco.ios](https://docs.ansible.com/ansible/latest/collections/cisco/ios/index.html)
 module in our playbook.  Three basic modules exist for interacting with Cisco IOS devices, ios_command, ios_config, and ios_facts.

Playbooks are saved into the appropriate folders in this directory structure.  Review the playbook for complete syntax 
and execute using the commands listed below.

playbooks are executed using the following command structure:

```
ansible-playbook <<group_name>> -i <<inventory_file>> <<playbook_name>>
``` 

### ios_command/nxos_command
 [ios_command](https://docs.ansible.com/ansible/latest/collections/cisco/ios/ios_command_module.html) and 
 [nxos_command](https://docs.ansible.com/ansible/latest/collections/cisco/nxos/nxos_command_module.html)
  allows us to send arbitrary commands to the device and return results.  We can specify wait_for behavior to list conditions 
  before moving forward with a task.  This can be for a single line or other commands that require parent 
 configuration and then chid commands (e.g. acl name then acl entries). 
 
#### 01-run_show_version
Run a show version command on multiple devices

01-run_show_version/main.yml playbook text
```
---
- name: run show version on remote devices
  hosts: all
  gather_facts: no

  vars:
    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: cisco.ios.ios
    ansible_become: yes
    ansible_gecome_method: enable

  tasks:
    - name: run show version on remote devices
      cisco.ios.ios_command:
        commands: show version
```

Execute playbook
```
ansible-playbook -i inventory.yml 01-run_show_version/main.yml -u cisco -k
```

Notice when we ran the playbook we get information for both the IOS and the NXOS devices back and fail one device, 10.10.20.174.

We can specify specific groups or parents using the ```-l``` option.  See below for a few examples of running the command 
against different inventory groups.

```
ansible-playbook -l devnet 01-run_show_version/main.yml
ansible-playbook -l devnet_ios 01-run_show_version/main.yml
ansible-playbook -l devnet_nxos 01-run_show_version/main.yml
```

#### 02-run_show_arp

This playbook will get the 'show arp' info from all devices in the inventory file.

```
ansible-playbook 02-show_arp/main.yml 
``` 


#### 03-run_show_mac_address_address-table

```
ansible-playbook 03-show_run_save_to_disk/main.yml 
``` 

#### 04-save_multiple_commands_to_text
This playbook will save multiple commands to a text file using the ios_command modules.  Two versions have been created.  
The file main.yml is targeted to IOS devices.  The file main_nxos.yml is targeted to NXOS devices.
```
ansible-playbook 04-save_multiple_commands_to_text/main.yml 
``` 

#### 05-run_commands_that_require_prompt
This playbook will run a command that typically requires user input before continuing.  Ansible will answer the prompt 
per the playbook specifications.
```
ansible-playbook 05-run_commands_that_require_prompt/main.yml 
``` 

### ios_config/nxos_config
  [ios_config](https://docs.ansible.com/ansible/latest/collections/cisco/ios/ios_config_module.html) and 
  [nxos_config](https://docs.ansible.com/ansible/latest/collections/cisco/nxos/nxos_config_module.html)
  allows us to send arbitrary configurations to the device and return results.  Consider this command as going into ```config t```
  mode and running any command you want.  Parent commands like interface or routing process can be specified seperately from 
  children commands.  Anything from hostname, NTP, SNMP, ACLs, OSPF, Interface, Routing Protocol, QoS, etc can be configured 
  using the ios_config module.  Look at the examples below and refer to official documentation linked above for a better
   understanding of how to use this module.  
  
#### 101-configure_loopback_setting
This playbook will configure Loopback 10 interface using variables stored in the host_var directory.
```
ansible-playbook 101-configure_loopback_setting/main.yml 
``` 
#### 102-configure_helpers_on_multiple_interfaces
This playbook will configure helper addresses on multiple interfaces.  Helper and specified interfaces are configured in 
the group_vars directory and only apply to the NXOS devices for this example.  
```
ansible-playbook 102-configure_helpers_on_multiple_interfaces/main.yml 
``` 

#### 103-configure_new_acl
This playbook will configure a new ACL called VTY_ACL and put all RFC1918 in the ACL.  Parent commands specify the top level 
command with the individual lines being entered under the parent.  Notice the match: exact.  If the ACL doesn't match exactly 
(including sequence numbers), ansible will execute the 'before' line and then reapply the configuration.  This ensures your ACLs
 always match your intent and golden config.
```
ansible-playbook 103-configure_new_acl/main.yml 
``` 
 
#### 104-compare_startup_to_running_config



### ios_facts/nxos_facts
  [ios_facts](https://docs.ansible.com/ansible/latest/collections/cisco/ios/ios_facts_module.html) and 
  [nxos_facts](https://docs.ansible.com/ansible/latest/collections/cisco/nxos/nxos_facts_module.html) allow us to pull 
  generic details about the device.  Every platform allows a bit different facts to be retrieved but typically allow things like 
  version information, hardware details, interfaces, configurations, etc.  The examples below show how to use the basic 
  facts module as well as gather a subset of the facts, exclude specific facts, or gather facts and save to files using 
  jinja2 templates.
  
#### 201-gather_legacy_facts
This is a basic playbook to show how to use the individual tasks to run gather_facts.  This would be useful if you didn't 
want to run gather_facts against all devices or if you wanted something more than the minimum (default) gather_facts. 
```
ansible-playbook 201-gather_legacy_facts/main.yml 
``` 

#### 202-gather_subset_legacy_facts
This is a basic playbook to show how to get facts by category from devices. Supported tags are: config, hardware, interfaces, 
min, or all_facts)
```
ansible-playbook 202-gather_subset_legacy_facts/main.yml --tags config 
```

```
ansible-playbook 202-gather_subset_legacy_facts/main.yml --tags hardware 
``` 

```
ansible-playbook 202-gather_subset_legacy_facts/main.yml --tags interfaces
``` 

```
ansible-playbook 202-gather_subset_legacy_facts/main.yml --tags min 
``` 

```
ansible-playbook 202-gather_subset_legacy_facts/main.yml --tags all_facts 
```  

#### 203-exclude_subset_from_facts
Similiar to the previous example but this time we will exclude facts using the ! exclusion in the playbook.  
Use the following tags to limit playbook targets (interfaces, config)

```
ansible-playbook 203-exclude_subset_from_facts/main.yml
```

```
ansible-playbook 203-exclude_subset_from_facts/main.yml --tags interfaces 
``` 

```
ansible-playbook 203-exclude_subset_from_facts/main.yml --tags configs
``` 

#### 204-gather_network_resources_and_minimal_legacy
In addition to legacy facts discussed already, the facts can pull back more details including L3 configuration, interfaces, neighbors, 
etc.   In this example we will use the network_resources subcomponent of facts to pull back a subsection of the supported commands.  
Tags supported include interfaces, l3_interfaces, acls, ospf_interfaces, and all_resources. 

```
ansible-playbook 204-gather_network_resources_and_minimal_legacy/main.yml
```

```
ansible-playbook 204-gather_network_resources_and_minimal_legacy/main.yml --tags interfaces 
``` 

```
ansible-playbook 204-gather_network_resources_and_minimal_legacy/main.yml --tags l3_interfaces
``` 

```
ansible-playbook 204-gather_network_resources_and_minimal_legacy/main.yml --tags acls
``` 


```
ansible-playbook 204-gather_network_resources_and_minimal_legacy/main.yml --tags ospf_interfaces
``` 


```
ansible-playbook 204-gather_network_resources_and_minimal_legacy/main.yml --tags all_resources
``` 

#### 205-gather_facts_save_using_jinja2_templates
Two main actions happen in this playbook.  Facts are gathered and then stored to a file in a new directory called facts.  
One file is based on a jinja2 template stored inline in our playbook.  The other uses a template in the /templates/ directory 
of the playbook.  In addition a backup is made of the configuration and stored in a new folder called /backups/ with each 
file being uniquely named after the hostname. 

```
ansible-playbook 205-gather_facts_save_using_jinja2_templates/main.yml
```