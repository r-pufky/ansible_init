# Init (Provision)
Init and provision machines for ansible management.

## [Requirements][i]
Requires [r_pufky.deb][g] galaxy-ng collection.

A user with **root** privileges (ability to modify users, groups, sudo,
services) with remote access to the system.

## Role Variables
Detailed variable use documented in defaults. See usage for role operation.

* [defaults][j] - User configurable options.

## Usage
Assumes target machine is a fresh install. This role will update the system,
create required users, and setup SSH/SSHD for future ansible management via
standard playbooks.

See defaults (and related role defaults) for in-depth documentation.

### Warning
> Specific user modifications on account that is being used for init may cause
> non-deterministic behavior while role is executing.
>
> If provisioning a machine for the first time, ensure that the **user**
> connected to run `r_pufky.deb.init` is not the user being added or modified.
>
> The provisioning user can be removed with standard user management in
> `r_pufky.deb.users` after init.

**HIGHLY** recommend init adds an ansible user via root or the first user.
Leave all other user modification to other roles via the ansible user. Many
pre-installed accounts will not have sudo access by default.

### Feature Flags
Tasks are gated by feature flags and executed in the following order.

  Step | Flag                   | Notes
 ------|------------------------|-------
  1    | init_flg_msg_provision | Display already provisioned messages.
  2    | init_flg_dist_upgrade  | Enable dist-upgrade during init.
  3    | init_flg_packages      | Install minimum ansible packages.
  4    | init_flg_users         | Provision system accounts.
  5    | init_flg_ssh           | Configure SSH/SSHD.

### Example Playbooks

#### Create ansible system user.
group_vars/all/users/ansible.yml
``` yaml
# Ansible account for all hosts.
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
  password: '*'  # see debian SSH pubkey changes.
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

# Setup ansible user during init.
init_cfg_users:
  - 'ansible'
```

#### Configure global SSH/SSHD settings.
group_vars/all/ssh.yml
``` yaml
# Set SSH/SSHD configuration for all systems.
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
See [r_pufky.deb.ssh][k] for detailed SSH/SSHD configuration.

#### Provision Playbook
Create plays specifically for **root** user **su** and **sudo** provisioning.

init.yml
``` yaml
- name: 'Init | root'
  hosts: 'all'
  remote_user: 'root'
  roles:
    - 'r_pufky.deb.init'
  tags:
    - 'root'
    - 'never'

- name: 'Init | user su'
  hosts: 'all'
  become: true
  become_method: 'ansible.builtin.su'
  roles:
    - 'r_pufky.deb.init'
  tags:
    - 'su'
    - 'never'

- name: 'Init | user sudo'
  hosts: 'all'
  become: true
  become_method: 'ansible.builtin.sudo'
  roles:
    - 'r_pufky.deb.init'
  tags:
    - 'sudo'
    - 'never'
```

``` bash
# Init using root account.
ansible-playbook init.yml --tags root --limit {HOST} --user root --ask-pass

# Init using {USER} account with su
ansible-playbook init.yml --tags su --limit {HOST} --user {USER} --ask-pass --ask-become-pass

# Init using {USER} account with sudo
# ssh-copy-id -i {USER}/host_ssh_key.pub {USER}@{HOST}  # if pubkey enabled
ansible-playbook init.yml --tags user --limit {HOST} --user {USER} --ask-pass --ask-become-pass
```

## Development
Configure [environment][a].

``` bash
# Run all tests.
molecule test --all
```

### [Releases][b]

  Release | Debian | Ansible | Notes
 ---------|--------|---------|-------
  3.x.x   | 13     | 2.20    | Ansible 2.20, feature flags, and semantic versioning.
  2.x.x   | 13     | 2.18    | Migrate to Debian Trixie.
  1.x.x   | 12     | 2.18    | Use standardized libraries.
  0.x.x   | 12     | 2.18    | Migration from private repository.

## Issues
Create a bug and provide as much information as possible.

Associate pull requests with a submitted bug.

## License
[AGPL-3.0 License][c] | [direct link][f]

## Author Information
PGP: [466EEC2B67516C7117C85CE3A0BC35D16698BAB9][d] | [github gist][e]

[a]: https://r-pufky.github.io/ansible_docs
[b]: https://semver.org/spec/v2.0.0
[c]: https://www.tldrlegal.com/license/gnu-affero-general-public-license-v3-agpl-3-0
[d]: https://keys.openpgp.org/vks/v1/by-fingerprint/466EEC2B67516C7117C85CE3A0BC35D16698BAB9
[e]: https://gist.github.com/r-pufky/a8df36977c55b5bb20829267c4c49d22

[f]: https://github.com/r-pufky/ansible_init/blob/main/LICENSE
[g]: https://github.com/r-pufky/ansible_collection_deb
[i]: https://github.com/r-pufky/ansible_init/blob/main/meta/main.yml
[j]: https://github.com/r-pufky/ansible_init/tree/main/defaults/main/main.yml
[k]: https://github.com/r-pufky/ansible_ssh/blob/main/defaults/main
