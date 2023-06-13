# Requirements

1. Ansible installed:

```bash
sudo apt install python3
python3 -m ensurepip --upgrade
pip3 install ansible
```

2. Requirements.yaml installed (this role uses [pimvh.ssh_keygen](https://github.com/pimvh/ssh_keygen)):

```bash
ansible-galaxy install -r requirements.yaml
```

## Required variables

Review the variables as shown in defaults.

```yaml
cloud_init_machine_name: ""
cloud_init_ansible_user_passwd_hash: "" # the hash of the password for the ansible user
cloud_init_github_token: ""

cloud_init_userdata:
  hostname: hostname
  fqdn: hostname.example.com
  groups: []
  users:
    - name: my user
      gecos: My user description
      shell: /bin/bash
      sudo: ALL=(ALL) NOPASSWD:ALL # passwordless sudo
      groups: sudo                 # member of sudo
      lock_passwd: false           # unlock password
      passwd: "{{ password_here |  password_hash('sha512') }}"
      ssh_authorized_keys: []      # optional authorized key
  runcmd: []                       # additional command to run in cloudinit
  writefiles: []                   # additional files to write
  packages: []                     # additional packages to install

cloud_init_networkdata:
    # define IPs and use the `default routes` and `nameservers` below
    ipv4: << ipv4 >>
    ipv6: << ipv6 >>
    # --- OR ---
    # dump an entire netplan
    # like the following
    netplan:
      network:
        version: 2
        ethernets:
          enp1s0:
            dhcp4: false
            addresses:
              - << addr >>
            gateway4: << addr >>
            gateway6: << addr >>
            nameservers:
              addresses:
              - << dns_server ip >>

cloud_init_netplan_routes:
  - to: default
    via: 1.0.0.1
  - to: default
    via: 2001:db8::11

cloud_init_netplan_nameservers:
  addresses:
    - 1.1.1.1
    - 1.0.0.1

cloud_init_add_to_known_hosts: true
cloud_init_reboot_on_finish: true
cloud_init_enable_ssh_ca: true

# My recommendation is to use lookup plugins like this:
# cloud_init_ssh_host_ca_publickey: "{{ lookup('ansible.builtin.file', 'your_ca') }}"
# or vars lookups like this:
# "{{ lookup('ansible.builtin.vars', 'your_ca') }}"
cloud_init_ssh_host_ca_privatekey: ""
cloud_init_ssh_host_ca_privatekey_pass: ""
cloud_init_ssh_host_ca_publickey: ""
cloud_init_ssh_user_ca_publickeys: []

cloud_init_enable_ansible_pull: false
cloud_init_ansible_pull_repo_owner: ""
cloud_init_ansible_pull_repo_name: ""
cloud_init_ansible_pull_playbook_name: ""
cloud_init_ansible_pull_deploy_key_name: "Ansible-pull deploy key"

cloud_init_validity_period: 520w
cloud_init_ssh_ca_runcmd:
  # configure ca usage on the server
  - echo "@cert-authority * $(cat /etc/ssh/host_ca.pub)" >> /etc/ssh/ssh_known_hosts
  # just remove the host CAs public key
  - rm -f /etc/ssh/host_ca.pub
  # configure new trustedUserCAkey key by appending the create cloud-init ssh config
  - echo "TrustedUserCAKeys /etc/ssh/ssh_trusted_user_ca_keys" >> /etc/ssh/sshd_config.d/50-cloud-init.conf
  # restart sshd to make changes have effect
  - systemctl restart sshd
```

# Example playbook

```yaml
hosts:
  - foo
roles:
  - pimvh.cloud_init

```

# TLDR - What will happen if I run this

- Assert required variables are defined
- Create directory to start cloud-init files
- Fetch Github hostkeys (when requested)
- Generate SSH key pair for Ansible pull (when requested)
- Configure SSH Host CAs and User CAs (when requested)
  - Sign certificate keys on the controller
- Configure Ansible-pull by putting requirements.yaml of the repo_url in the cloud-int config (when requested)
- Template cloud_init
  - Template signed certificates to cloud-init configuration
  - Template required cloud-init userdata
  - Template cloud-init networkdata
  - Template Ansible pull requirements.yaml
- Add Github deploy key to requested Ansiblepull repository (when requested)
- Run Ansible-pull as the ansible user on the system (when requested)
- Add SSH CA to known hosts (when requested)

# Future Improvements

- Find a better way to get shortlived access to Github using different auth method.

# Sources

Part of the SSH CA logic is based on the [following blog](https://www.thiswayup.de/blog/2021/ssh-host-key-ca-with-cloud-init-and-terraform.html)
