# Ansible Demo

## Installation

```bash
brew install ansible
```

OR

```bash
pip3 install ansible
```

### Check the installed Version
```bash
ansible --version
```

## Create Ansible Inventory
The file should be created under the path: /etc/ansible/hosts

```yaml
all:
  hosts:
    bastion:
       ansible_host: 129.153.60.16
  children:
    webservers:
      hosts:
        web1:
           ansible_host: 192.18.154.93
        web2:
           ansible_host: 150.230.31.141
    dbservers:
      hosts:
        database1:
           ansible_host: 140.238.159.28
        database2:
           ansible_host: 129.153.51.158
    ashburn:
      hosts:
        web1:
        database1:
    phoenix:
      hosts:
        web2:
        database2:
    prod:
      children:
        ashburn:
    dr:
      children:
        phoenix:
```

verify Inventory
```bash
ansible-inventory --list
```

[How to build your inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)


## Ansible Adhoc Commands

```bash
ansible localhost -m ping

ansible webservers -m ping  # Should result in error

# Disable host checking
ANSIBLE_HOST_KEY_CHECKING=False ansible all -m ping  # Error again

ansible all -m ping -u opc

# Condensed Output
ansible all -m ping -u opc -o

# Control the Ansible Fork
ansible all -m ping -u opc -o -f 1

ansible all -m shell -a uptime -u opc

ansible all -m shell -a "free -m" -u opc

ansible all -m shell -a "uname -r" -u opc

ansible all -m shell -a "cat /etc/redhat-release" -u opc

ansible dbservers -m copy -a "src=~/install_media.zip dest=/tmp"

ansible webservers -m yum -a "name=httpd state=present" -u opc # Results in error

## Become a privileged user
ansible webservers -m yum -a "name=httpd state=present" -u opc -b

# Delete the package for now, will reinstall using playbook
ansible webservers -m yum -a "name=httpd state=absent" -u opc -b

# Get Ansible Facts
ansible web1 -m setup -u opc

ansible all -m setup -a "filter=ansible_all_ipv4_addresses" -u opc

# Verbose Output
ansible all -m shell -a "cat /etc/redhat-release" -u opc -vv # or use -v or -vvv


# Running ansible command against a target not defined in inventory
ansible all -i 150.230.31.141, -m shell -a hostname -u opc
```

## Getting help on a Module
```bash
ansible-doc -l

ansible-doc -s yum
```


## Ansible Playbooks

```yaml
---
- name: Update web servers
  hosts: all
  remote_user: opc
  become: true


  tasks:
  - name: Ensure apache is at the latest version
    ansible.builtin.yum:
      name: httpd
      state: latest

  - name: Start the apache server
    ansible.builtin.systemd:
      name: httpd
      state: started
      enabled: true

  - name: Enable Firewalld for HTTP
    ansible.posix.firewalld:
      service: http
      permanent: yes
      state: enabled

  - name: Reload Firewalld 
    ansible.builtin.shell:
      cmd: systemctl reload firewalld
```

```bash
ansible-playbook -l webservers web.yaml
```

### Ansible Loops
```yaml
---
- hosts: all
  remote_user: opc
  become: yes
  become_user: root
  gather_facts: false
  tasks:
  - name: Install OCI CLI / additional RPM packages
    ansible.builtin.yum:
      name: "{{ item }}"
      state: latest
    loop:
      - oraclelinux-developer-release-el8
      - python36-oci-cli
      - stress-ng
      - telnet
      - git
      - xorg-x11-xauth
      - xterm
      - pip


  - name: Add helm Chart repositories
    kubernetes.core.helm_repository:
      name: "{{ item.name }}"
      repo_url: "{{ item.url }}"
    loop:
      - {name: prometheus-community, url: https://prometheus-community.github.io/helm-charts}
      - {name: elastic,   url: https://helm.elastic.co}
      - {name: grafana,   url: https://grafana.github.io/helm-charts}
      - {name: hashicorp, url: https://helm.releases.hashicorp.com}
      - {name: bitnami,   url: https://charts.bitnami.com/bitnami}
      - {name: datadog,   url: https://helm.datadoghq.com}
      - {name: jetstack , url: https://charts.jetstack.io}
```

### Ansible Facts in Playbook
```yaml
---
- hosts: all
  become: true
  become_user: root
  tasks:

  - name: Update all Packages on yum based systems
    yum:
       name:
          - tcl
          - redis_exporter
       update_cache: true
       state: latest
    when: ansible_distribution in ["OracleLinux", "CentOS"]


  - name: Update all Packages on apt based systems
    apt:
       name: net-tools
       update_cache: true
       state: latest
    when: ansible_distribution in ["Ubuntu", "Debian"]
```


### Ansible Variables
```yaml

##  main.yaml
---
- hosts: all
  gather_facts: false
  remote_user: ansadm
  become: true
  become_user: root
  tasks:

- ansible.builtin.import_playbook: kubernetes.yaml
  vars:
     VERSION: "{{ version }}"

- ansible.builtin.import_playbook: controlplane.yaml
  vars:
     PodCIDR: 10.244.0.0/16
     APIaddress: 192.168.1.171


## kubernetes.yaml
---
- hosts: all
  gather_facts: false
  become: true
  become_user: root

  tasks:    
    - name: Install Kubernetes packages
      vars:
       version:
      yum:
        name: "{{ item }}"
        state: latest
        disable_excludes: kubernetes
      loop:
        - kubelet-{{ VERSION }}
        - kubeadm-{{ VERSION }}
        - kubectl-{{ VERSION }}


## ccontrolplane.yaml
---
- hosts: masters
  gather_facts: false
  become: true
  become_user: root

  tasks:
    - name: Setup ControlPlane (execute on master only)
      shell:
        cmd: kubeadm init --pod-network-cidr "{{ PodCIDR }}" --apiserver-advertise-address="{{ APIaddress }}"
      register: execution_status
```

```bash
ansible-playbook -l kubernetes -e version=1.26.3 ~/Install/main.yaml
```



## Ansible OCI Module
Galaxy provides pre-packaged units of work known to Ansible as roles and collections.

```bash
ansible-galaxy -h

# View Installed collection / modules
ansible-galaxy collection list

# Install OCI module for ansible
ansible-galaxy collection install oracle.oci

# Periodically upgrade the OCI module for ansible
ansible-galaxy collection install oracle.oci --upgrade
```
[Oracle Cloud Infrastructure Ansible Collection](https://galaxy.ansible.com/oracle/oci)


## Ansible OCI Module in Action
Notice the value of hosts parameter

```yaml
---
- hosts: localhost
  gather_facts: false
  collections:
    - oracle.oci
  vars:
    bucket_name: ansible_bucket
    tenancy_namespace: orasenatdpltintegration03
    compartment_ocid: ocid1.compartment.oc1..aaaaaaaaebfcvucovsujhajrikjge63w333hirsdsop7yrr6uehti24kecpa
  tasks:
    - block:

      - name: Create bucket
        oci_object_storage_bucket:
          # required
          compartment_id: "{{ compartment_ocid }}"
          namespace_name: "{{ tenancy_namespace }}"
          name: "{{ bucket_name }}"
          # optional
          storage_tier: Standard
          metadata: null
          public_access_type: NoPublicAccess
          object_events_enabled: true
          freeform_tags: {'Demo': 'Ansible'}
          versioning: Enabled
```

```bash
    ansible-playbook -i localhost oci_bucket.yaml
```

[Oracle Cloud Infrastructure Ansible Collection](https://oci-ansible-collection.readthedocs.io/en/latest/)

## Best Practice
Break the ansible code by functionality into smaller files, and reference those using a single file

```yaml
---
- import_playbook: misc.yaml
- import_playbook: helm.yaml
- import_playbook: mysql.yaml
- import_playbook: splunk.yaml
- import_playbook: postgreSQL.yaml
- import_playbook: datadog.yaml
```

## Links
[Introduction to ad hoc commands](https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html)

[Ansible playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)

[Working with playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks.html)

[](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)
