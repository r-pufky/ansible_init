# Init (Provision)
Init and provision machines for ansible management.

## Requirements
[supported platforms](https://github.com/r-pufky/ansible_init/blob/main/meta/main.yml)

[collections/roles](https://github.com/r-pufky/ansible_init/blob/main/meta/requirements.yml)

## Role Variables
[defaults](https://github.com/r-pufky/ansible_init/tree/main/defaults/main.yaml)

## Dependencies
A user with **root** privileges (ability to modify users, groups, sudo, services)
with remote access to the system.

## Example Playbook
Assumes target machine is a fresh install. This role will update the system,
create required users, and setup SSH/SSHD for future ansible management via
standard playbooks.

See defaults (and related role defaults) for in-depth documentation.

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
  password: '!'
  password_lock: true
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
ssh_server_use_pam: true
ssh_server_allow_agent_forwarding: false
ssh_server_print_motd: false
ssh_server_subsystem:
  - system: 'sftp'
    config: 'internal-sftp'
ssh_server_allow_groups: ['ssh']
```

### Provision Playbook
Create plays specifically for **root** user and **user** provisioning. Force
use of `provision_*` tag for target provisioning.

provision.yml
``` yaml
- name: 'Provision hosts via root user'
  hosts: 'all'
  remote_user: 'root'
  tasks:
    - name: 'Provision hosts via root user'
      ansible.builtin.include_role:
        name: 'r_pufky.srv.init'
  tags:
    - 'provision_root'
    - 'never'

- name: 'Provision hosts via user with sudo'
  hosts: 'all'
  become: true
  tasks:
    - name: 'Provision hosts via user with sudo'
      ansible.builtin.include_role:
        name: 'r_pufky.srv.init'
  tags:
    - 'provision_user'
    - 'never'
```

#### Provisioning with ROOT account
``` bash
ansible-playbook provision.yml --tags provision_root --limit {HOST} --ask-pass
```

#### Provisioning with USER account with SUDO access

Copy user public key over if needed.
``` bash
ssh-copy-id -i {USER}/host_ssh_key.pub {USER}@{HOST}
```

Provision
``` bash
ansible-playbook provision.yml --tags provision_user --limit {HOST} --user {USER} --ask-become-pass
```

## Unit Testing
Test framework requires molecule and rootless podman setup.

Run all unit tests:
``` bash
molecule test --all
```

## Issues
Create a bug and provide as much information as possible.

Associate pull requests with a submitted bug.

## License
[AGPL-3.0 License](https://www.tldrlegal.com/license/gnu-affero-general-public-license-v3-agpl-3-0)
 [(direct link)](https://github.com/r-pufky/ansible_users/blob/main/LICENSE)

## Author Information
PGP Fingerprint: [466EEC2B67516C7117C85CE3A0BC35D16698BAB9](https://keys.openpgp.org/vks/v1/by-fingerprint/466EEC2B67516C7117C85CE3A0BC35D16698BAB9)
| [github gist](https://gist.github.com/r-pufky/a8df36977c55b5bb20829267c4c49d22)

