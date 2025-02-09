# Init (Provision)
Init and provision machines for ansible management.

## Requirements
[supported platforms](https://github.com/r-pufky/ansible_init/blob/main/meta/main.yml)

[collections/roles](https://github.com/r-pufky/ansible_init/blob/main/meta/requirements.yml)

## Role Variables
[defaults](https://github.com/r-pufky/ansible_init/tree/main/defaults/main.yaml)

## Dependencies
Part of the [r_pufky.srv](https://github.com/r-pufky/ansible_collection_srv)
collection.

A user with **root** privileges (ability to modify users, groups, sudo, services)
with remote access to the system.

## Example Playbook
Assumes target machine is a fresh install. This role will update the system,
create required users, and setup SSH/SSHD for future ansible management via
standard playbooks.

See defaults (and related role defaults) for in-depth documentation.

### Warning
> Specific user modifications on account that is being used for init may cause
> non-deterministic behavior while role is executing.
>
> If provisioning a machine for the first time, ensure that the **user**
> connected to run `r_pufky.srv.init` is not the user being added or modified.

**HIGHLY** recommend init adds an ansible user via root or the first user.

Leave all other user modification to other roles via the ansible user. Many
pre-installed accounts will not have sudo access by default.

### Ansible Account
Define the ansible account for all hosts and specify that user to be created
with the init/provisioning role.


group_vars/all/users/ansible.yml
``` yaml
users_group_ansible:
  name: 'ansible'
  gid: 900

users_user_ansible:
  name: 'ansible'
  group: '{{ users_group_ansible.name }}'
  groups:
    - '{{ users_group_ansible.name }}'
    - 'sudo'
  uid: 900
  home: '/etc/ansible/.user'
  shell: '/bin/bash'
  create_home: true
  password: '*'  # see debian SSH pubkey changes
  update_password: 'always'
  expires: -1
  system: true
  extensions:
    include_group: true
    sudoers:
      commands: 'ALL'
      name: 'ansible'
      nopassword: true
      state: 'present'
      user: 'ansible'
      runas: 'ALL'
      validation: 'detect'
    ssh:
      authorized_keys: '{{ vault_ansible_authorized_keys }}'

init_accounts_users:
  - 'ansible'
```
### SSH/SSHD Management
See [r_pufky.srv.ssh](https://github.com/r-pufky/ansible_ssh/blob/main/defaults/main/)
for detailed SSH/SSHD configuration.

Secure SSHD and configure SSH for all systems.

group_vars/all/ssh.yml
``` yaml
ssh_security_remove_host_keys:
  - 'ssh_host_dsa_key'
  - 'ssh_host_ecdsa_key'
ssh_security_regenerate_host_rsa_keys: 4096

ssh_server_host_key:
  - '/etc/ssh/ssh_host_rsa_key'
  - '/etc/ssh/ssh_host_ed25519_key'
ssh_server_permit_root_login: false
ssh_server_ignore_user_known_hosts: true
ssh_server_password_authentication: false
ssh_server_allow_agent_forwarding: false
ssh_server_print_motd: false
ssh_server_allow_groups: ['_ssh']
```

### Provision Playbook
Create plays specifically for **root** user **su** and **sudo** provisioning.

init.yml
``` yaml
- name: 'Init | root'
  hosts: 'all'
  remote_user: 'root'
  roles:
    - 'r_pufky.srv.init'
  tags:
    - 'root'
    - 'never'

- name: 'Init | user su'
  hosts: 'all'
  become: true
  become_method: 'ansible.builtin.su'
  roles:
    - 'r_pufky.srv.init'
  tags:
    - 'su'
    - 'never'

- name: 'Init | user sudo'
  hosts: 'all'
  become: true
  become_method: 'ansible.builtin.sudo'
  roles:
    - 'r_pufky.srv.init'
  tags:
    - 'sudo'
    - 'never'
```

#### Init using root account
``` bash
ansible-playbook init.yml --tags root --limit {HOST} --user root --ask-pass
```

#### Init using {USER} account with su
``` bash
ansible-playbook init.yml --tags su --limit {HOST} --user {USER} --ask-pass --ask-become-pass
```

#### Init using {USER} account with sudo
``` bash
# ssh-copy-id -i {USER}/host_ssh_key.pub {USER}@{HOST}  # if pubkey enabled
ansible-playbook init.yml --tags user --limit {HOST} --user {USER} --ask-pass --ask-become-pass
```

## Development
Configure [environment](https://github.com/r-pufky/ansible_collection_srv/blob/main/docs/dev/environment/README.md)

Run all unit tests:
``` bash
molecule test --all
```

### Issues
Create a bug and provide as much information as possible.

Associate pull requests with a submitted bug.

## License
[AGPL-3.0 License](https://www.tldrlegal.com/license/gnu-affero-general-public-license-v3-agpl-3-0)
 [(direct link)](https://github.com/r-pufky/ansible_init/blob/main/LICENSE)

## Author Information
PGP Fingerprint: [466EEC2B67516C7117C85CE3A0BC35D16698BAB9](https://keys.openpgp.org/vks/v1/by-fingerprint/466EEC2B67516C7117C85CE3A0BC35D16698BAB9)
| [github gist](https://gist.github.com/r-pufky/a8df36977c55b5bb20829267c4c49d22)
