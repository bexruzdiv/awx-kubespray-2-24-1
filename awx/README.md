# How to modify kubespray for awx?

1. Change playbooks/reset.yml and playbooks/remove_node.yml
The purpose of this is to disable "yes" or "no" confirmation requests in the process of deleting nodes. Because the request cannot be answered through awx and the process freezes

Replace the playbooks/reset.yml file configuration with the following code
```
---
- name: Common tasks for every playbooks
  import_playbook: boilerplate.yml

- name: Gather facts
  import_playbook: facts.yml

- name: Reset cluster
  hosts: etcd:k8s_cluster:calico_rr
  gather_facts: False
  pre_tasks:
    - name: Log reset initiation
      debug:
        msg: "Initiating cluster reset without confirmation."

    - name: Gather information about installed services
      service_facts:

  environment: "{{ proxy_disable_env }}"
  roles:
    - role: kubespray-defaults
    - role: kubernetes/preinstall
      when: "dns_mode != 'none' and resolvconf_mode == 'host_resolvconf'"
      tags: resolvconf
      dns_early: true
    - role: reset
      tags: reset
```

Replace the playbooks/remove_node.yml file configuration with the following code

```
---
- name: Common tasks for every playbook
  import_playbook: boilerplate.yml

- name: Gather facts
  import_playbook: facts.yml
  when: reset_nodes | default(True) | bool

- name: Reset node
  hosts: "{{ node | default('kube_node') }}"
  roles:
    - { role: kubespray-defaults, when: "reset_nodes | default(True) | bool" }
    - { role: remove-node/pre-remove, tags: pre-remove }
    - { role: remove-node/remove-etcd-node }
    - { role: reset, tags: reset, when: "reset_nodes | default(True) | bool" }

# Note: Currently cannot remove the first master or etcd
- name: Post node removal
  hosts: "{{ node | default('kube_control_plane[1:]:etcd[1:]') }}"
  roles:
    - { role: kubespray-defaults, when: "reset_nodes | default(True) | bool" }
    - { role: remove-node/post-remove, tags: post-remove }
```


2. If you want to use `cilium` cni, the owner of the `/opt/cni/bin` directory should be changed to `root` to avoid the permission denied error during cilium installation
First change environment `kube_network_plugin: cni`. This is provide you build k8s cluster without cni 
Second change configuration of `roles/network_plugin/cni/tasks/main.yml`
Actually the configuration will be like this

![image](https://github.com/bexruzdiv/awx-kubespray-2-24-1/assets/107495220/e298b41d-6142-45b2-8d90-b74a847f1a20)

and you need to change `owner` to `root` as shown in the picture

![image](https://github.com/bexruzdiv/awx-kubespray-2-24-1/assets/107495220/00cfc743-def5-471b-a07c-e544cd81c5f2)

3. And finally comment your .gitignore file before pushing to the github repository

# How to setting AWX for kubespray
1. If you need, Create organization in awx 
2. You need to create credential for connect servers (in my case, i used ssh private key without passwords).

  - From left menu "Credentials" ➝ Add
  - Name  ➝  name for your crendential name
  - "Credential Type" ➝ Machine
  - "Organization" ➝ your organization
  - "SSH Private Key" ➝ your private ssh key, if a private key was used
  - "Privilege Escalation Method" ➝ sudo
  - "Privilege Escalation Username" ➝ root
  - "Save"

![image](https://github.com/bexruzdiv/awx-kubespray-2-24-1/assets/107495220/06d78578-f3f9-464a-a032-538c986a5131)


3. Create Inventoty
  - From left menu "Inventories" ➝ Add ➝ Add inventory
  - "Name"  ➝ name for your inventory 
  - "Organization" ➝ your organization
  - "Save"

![image](https://github.com/bexruzdiv/awx-kubespray-2-24-1/assets/107495220/9cbe9e90-d41f-4f1f-a426-40f131beb5da)

4. Create hosts
  -  From left menu "Hosts" ➝ Add
  -  "Name"  ➝ name for your host
  -  "Inventory"  ➝ your inventory
  -  
![image](https://github.com/bexruzdiv/awx-kubespray-2-24-1/assets/107495220/f6878f83-2765-4141-be05-7e35aa956d77)

5. Create groups in your inventory
    From left menu "Inventories" ➝ your inventory ➝ Groups ➝ Add 
    1. name: `kube_control_plane` and "Save" ➝ "Hosts"  ➝ "Add"  ➝ "Add existing host" ➝ choose your host for __master__ and "Save"
    2. name: `kube_node` and "Save" ➝  "Hosts" ➝ "Add"  ➝ "Add existing host" ➝ choose your host for __worker__ and "Save"
    3. name: `etcd` and "Save"  ➝  "Hosts" ➝ "Add"  ➝ "Add existing host" ➝ choose your host for __etcd__ and "Save"
    4. name: `k8s_cluster` and "Save" ➝ "Related Groups" ➝ "Add"  ➝ "Add existing group" ➝ chooce "kube_control_plane" and "kube_node" and "Save"
   
![image](https://github.com/bexruzdiv/awx-kubespray-2-24-1/assets/107495220/cf63e14d-13cd-4548-a6a3-d08a0b372165)

6. Create project
  -  From left menu "Projects"  ➝ Add
  -  "Name"  ➝  name for your project
  -  "Organization"  ➝  your organization
  -  "Source Control Type *"  ➝  "Git"
  -  "Source Control URL"   ➝  your kubespray url (github, gitlab)
  -  "Source Control Branch/Tag/Commit"   ➝  your branch
  -  "Options"    ➝   "Update Revision on Launch" and "Clean"

![image](https://github.com/bexruzdiv/awx-kubespray-2-24-1/assets/107495220/2cfa2129-2f53-46bc-88ba-3b6237ebb658)

7. Create templates
  -  From left menu "Templates"  ➝ "Add" ➝  "Add job template"
  -  "Name"  ➝  name for your templates
  -  "Inventory"  ➝ your inventory
  -  "Project"  ➝ your Project
  -  "Execution Environment"  ➝  your exucution environment (image)
  -  Playbook  ➝  cluster.yml (for create k8s cluster)
  -  Credentials  ➝  your crendential for access servers
  -  Variables for kubespray (like: `kube_network_plugin` or `loadbalancer_apiserver` etc... )
  -  "Save" and "Launch"

![image](https://github.com/bexruzdiv/awx-kubespray-2-24-1/assets/107495220/a85a4d6c-35cd-4c02-a0a9-40bec6cc2c45)


