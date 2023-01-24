# Requirements

1. Ansible installed:

```
sudo apt install python3
python3 -m ensurepip --upgrade
pip3 install ansible
```

2. Requirements.yaml installed (this role uses [pimvh.ssh_keygen](https://github.com/pimvh/ssh_keygen)):

```
ansible-galaxy install -r requirements.yaml
```

## Required variables

Review the variables as shown in defaults.

```
cloud_init_machine_name: ""
cloud_init_github_token: ""

cloud_init_userdata: {}
# cloud_init_userdata:
#   hostname: hostname
#   fqdn: hostname.example.com
#   groups: []
#   users:
#     - name: my user
#       gecos: My user description
#       shell: /bin/bash
#       sudo: ALL=(ALL) NOPASSWD:ALL # passwordless sudo
#       groups: sudo                 # member of sudo
#       lock_passwd: false           # unlock password
#       passwd: "pass_hash here"
#       ssh_authorized_keys:
#         - auth key                 # optional authorized key
#   runcmd: []                       # command to run in cloudinit
#   writefiles: []                   # files to write
#   packages: []                     # packages to install
# cloud_init_networkdata: {}
cloud_init_networkdata: {}
# cloud_init_networkdata:
#     # define IPs
#     ipv4: << ipv4 >>
#     ipv6: << ipv6 >>
#     # --- OR ---
#     # dump an entire netplan
#     # like the following
#     netplan:
#       network:
#         version: 2
#         ethernets:
#           enp1s0:
#             dhcp4: false
#             addresses:
#               - << addr >>
#             gateway4: << addr >>
#             gateway6: << addr >>
#             nameservers:
#               addresses:
#               - << dns_server ip >>

cloud_init_netplan_routes:
  []
  # - to: default
  #   via: 1.0.0.1
  # - to: default
  #   via: 2001:db8::11

cloud_init_netplan_nameservers:
  {}
  # addresses:
  #   - 1.1.1.1
  #   - 1.0.0.1

cloud_init_register_github_key: true
cloud_init_add_to_known_hosts: true
cloud_init_reboot_on_finish: true
cloud_init_enable_ssh_ca: true

# Use lookup plugins like this:
# cloud_init_ssh_host_ca_publickey: "{{ lookup('ansible.builtin.file', 'os3_user_ca.pub') }}"
cloud_init_ssh_host_ca_privatekey: ""
cloud_init_ssh_host_ca_privatekey_pass: ""
cloud_init_ssh_host_ca_publickey: ""
cloud_init_ssh_user_ca_publickeys: []

cloud_init_enable_ansible_pull: false
cloud_init_ansible_pull_url: ""
cloud_init_ansible_playbook_name: ""

cloud_init_validity_period: 520w
cloud_init_ssh_ca_runcmd:
  # sign and configure ssh host keys
  - ssh-keygen -s /etc/ssh/host_ca -I "$(hostname)" -n "$(hostname),$(hostname --fqdn)" -V -5m:+{{ cloud_init_validity_period }} -h -P "$(cat /etc/ssh/host_ca_pass)" /etc/ssh/ssh_host_ed25519_key.pub
  # remove ssh key signing private key and password file
  - rm -f /etc/ssh/host_ca /etc/ssh/host_ca_pass
  # configure ca usage on the server
  - echo "@cert-authority * $(cat /etc/ssh/host_ca.pub)" >> /etc/ssh/ssh_known_hosts
  # configure new signed host certificates
  - echo "HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub" >> /etc/ssh/sshd_config
  - echo "TrustedUserCAKeys /etc/ssh/ssh_trusted_user_ca_keys" >> /etc/ssh/sshd_config
  # restart sshd to make changes have effect
  - systemctl restart sshd
```

# Example playbook

```
hosts:
  - foo
roles:
  - pimvh.cloud_init

```

# TLDR - What will happen if I run this

- Assert required variables are defined
- Create directory to start cloud_init files
- Fetch Github hostkeys (when requested)
- Generate Github SSH-key (when requested)
- Configure SSH Host CAs and User CAs (when requested)
- Template Cloud_init userdata
- Template Cloud_init networkdata
- Register Github SSH-key (when requested)
- Add SSH CA to known hosts (when requested)
- Add SSH Host CA to known hosts (when requested)

# Future Improvements

- Consider a different way of signing hostkey, by creating cert beforehand (circumvents adhoc signing of key)
- Make key generation non interactive

# Sources

Part of the SSH CA logic is based on the [following blog](https://www.thiswayup.de/blog/2021/ssh-host-key-ca-with-cloud-init-and-terraform.html)
