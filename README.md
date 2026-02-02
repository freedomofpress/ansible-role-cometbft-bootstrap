# CometBFT bootstrap Ansible role

## What is this?

This repository holds an Ansible role designed to generate a CometBFT
testnet using the local machine as the bootstrap node (to generate
keys and so on).

The testnet is bootstrapped using Docker on the local workstation
executing the Ansible playbook.

It then is designed to sync that configuration out to the actual
testnet nodes, which presumably are also Ansibilized using our
accompanying Ansible role, [ansible-role-cometbft](https://github.com/freedomofpress/ansible-role-cometbft).

## What's the testnet setup?

We assume 3 validator nodes and 3 sentry nodes.

Each validator can only receive traffic from the sentries.

The sentries can receive traffic from all sentries and validators.

## How do I run the Ansible role?

Below are example playbooks. It assumes that your inventory contains:

 * A hostgroup called `cometbft_bootstrap` that most likely has just
   `localhost` in it

 * A hostgroup called `cometbft_validators` that has the 3 validator
   node IPs/hostnames in it

 * A hostgroup called `cometbft_sentries` that has the 3 sentry node
   IPs/hostnames in it

 * Optionally a `cometbft` group that includes all three (to share
   common inventory attributes)

The inventory attributes for each group:

### cometbft group
```
# Version
cometbft_version: "v0.38.21"

# Where the node home lives on each machine
cometbft_home: "/opt/cometbft"

# Where to build one-time testnet config on the control node (localhost, usually)
cometbft_testnet_dir: "/tmp/comet-testnet"

# Docker image
cometbft_docker_image: "cometbft/cometbft:{{ cometbft_version }}"
```

### cometbft_validators group
```
cometbft_role: "validator"
```

### cometbft_sentries group
```
cometbft_role: "sentry"
```
 
The inventory attributes for each validator node would contain something
like this:

```
cometbft_node_index: 0 # incrementing for each other validator node
cometbft_partner: "my-sentry-1" # validator 2 would have sentry 2, etc
```

The inventory attributes for each sentry node would contain something
like this:

```
cometbft_node_index: 3
cometbft_partner: "my-validator-1"
```

### The playbook to bootstrap (using this role)

```
---
- name: Bootstrap CometBFT testnet config
  hosts: cometbft_bootstrap
  connection: local
  gather_facts: false
  roles:
    - ansible-role-cometbft-bootstrap

- name: Push node config to each remote node
  hosts: "cometbft_validators:cometbft_sentries"
  become: true
  tasks:
    - name: Ensure cometbft home exists
      ansible.builtin.file:
        path: "{{ cometbft_home }}"
        state: directory
        mode: "0755"

    - name: Sync node{{ cometbft_node_index }} dir to remote CMTHOME
      ansible.posix.synchronize:
        src: "{{ cometbft_testnet_dir }}/node{{ cometbft_node_index }}/"
        dest: "{{ cometbft_home }}/"
        delete: yes
```

### The playbook to configure (post-bootstrap, using ansible-role-cometbft)

```
---
- name: Set up CometBFT nodes
  hosts: "cometbft_validators:cometbft_sentries"
  become: true
  roles:
    - ansible-role-cometbft
```
