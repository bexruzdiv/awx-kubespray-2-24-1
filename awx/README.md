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
