# OpenStack 2025.1 (Epoxy) Automated Deployment

Complete automation for deploying OpenStack 2025.1 on a single KVM host with multiple VMs.

## Architecture

- **Control Node**: 1x (ctrl-01) - 16 vCPU, 64GB RAM, 200GB + 500GB disks
- **Compute Nodes**: 3x (cmp-01, cmp-02, cmp-03) - 8 vCPU, 32GB RAM, 100GB disk each
- **Networks**: Management (192.168.100.0/24), Storage (192.168.101.0/24), Provider (192.168.102.0/24)

## Prerequisites

- AlmaLinux/Rocky Linux 9+ as KVM host
- Minimum 96GB RAM
- Minimum 1TB storage
- Internet connectivity

## Quick Start

### 1. Complete Deployment (VMs + OpenStack + Validation)
```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

### 2. Step-by-Step Deployment
```bash
# Deploy VMs only
ansible-playbook -i inventory/hosts.ini deploy_vms.yml

# Deploy OpenStack only
ansible-playbook -i inventory/hosts.ini deploy_openstack.yml

# Validate deployment
ansible-playbook -i inventory/hosts.ini validate_openstack.yml
```

### 3. Destroy Everything
```bash
ansible-playbook -i inventory/hosts.ini destroy_all.yml
```

## Using tmux for Long-Running Deployments
```bash
# Start new tmux session
tmux new -s openstack

# Run deployment
ansible-playbook -i inventory/hosts.ini site.yml

# Detach from session (Ctrl+b, then d)
# Reattach later
tmux attach -s openstack
```

## Post-Deployment Access

### Horizon Dashboard
- URL: http://192.168.100.200
- Admin User: `admin`
- Admin Password: Check `/etc/kolla/passwords.yml` (keystone_admin_password)
- Demo User: `demo`
- Demo Password: `demo`

### Command Line Access
```bash
# SSH to control node
ssh ansible@192.168.100.100

# Source admin credentials
source /etc/kolla/admin-openrc.sh

# Run OpenStack commands
openstack server list
openstack network list
openstack compute service list
```

## Deployment Stages

1. **KVM Host Setup** (~5 minutes)
   - Install virtualisation packages
   - Configure nested virtualisation
   - Setup NTP server
   - Generate SSH keys

2. **VM Infrastructure** (~10 minutes)
   - Create libvirt networks
   - Create storage pools
   - Download base image
   - Create and boot VMs

3. **VM Prerequisites** (~15 minutes)
   - Wait for cloud-init
   - Install Docker
   - Configure networking
   - Setup NTP clients

4. **Kolla Bootstrap** (~10 minutes)
   - Install Kolla-Ansible
   - Generate configurations
   - Distribute SSH keys

5. **OpenStack Deployment** (~45-60 minutes)
   - Run prechecks
   - Bootstrap servers
   - Deploy OpenStack services
   - Run post-deploy

6. **OpenStack Initialization** (~10 minutes)
   - Create networks
   - Upload images
   - Create flavors
   - Setup demo project

7. **Validation** (~5 minutes)
   - Verify services
   - Test instance creation
   - Validate networking
   - Generate report

**Total Time**: ~90-120 minutes

## Troubleshooting

### Check deployment logs
```bash
ssh ansible@192.168.100.100
tail -f /var/log/kolla/ansible.log
```

### Verify services
```bash
ssh ansible@192.168.100.100
source /etc/kolla/admin-openrc.sh
openstack compute service list
openstack network agent list
docker ps
```

### Re-run specific stages
```bash
# Re-run only deployment
ansible-playbook -i inventory/hosts.ini deploy_openstack.yml --tags deploy

# Re-run only validation
ansible-playbook -i inventory/hosts.ini validate_openstack.yml
```

### Common Issues

**Issue**: MariaDB password mismatch
**Solution**: Ensure clean deployment - destroy VMs completely before redeploying

**Issue**: Compute nodes not registering
**Solution**: Check RabbitMQ connectivity and ensure HAProxy is configured correctly

**Issue**: OVS external_ids errors on compute nodes
**Solution**: These are non-critical and can be ignored (--skip-tags openvswitch is already applied)

## File Structure
```
/etc/ansible/
├── inventory/
│   └── hosts.ini
├── group_vars/
│   ├── all.yml
│   ├── control.yml
│   └── compute.yml
├── roles/
│   ├── 00_kvm_host_setup/
│   ├── 01_vm_infrastructure/
│   ├── 02_vm_prerequisites/
│   ├── 03_kolla_bootstrap/
│   ├── 04_kolla_deploy/
│   ├── 05_openstack_init/
│   └── 06_openstack_validate/
├── site.yml
├── deploy_vms.yml
├── deploy_openstack.yml
├── validate_openstack.yml
├── destroy_all.yml
├── ansible.cfg
└── README.md
```

## Customization

### Change VM specifications
Edit `group_vars/control.yml` and `group_vars/compute.yml`

### Change network ranges
Edit `group_vars/all.yml`

### Enable additional OpenStack services
Edit `roles/03_kolla_bootstrap/templates/globals.yml.j2`

### Change OpenStack version
Edit `group_vars/all.yml` - set `openstack_release` to desired version

## Validation Report

After deployment, find the validation report at:
```
/home/ansible/openstack-validation-report.txt
```

## Support

For issues or questions, review:
1. Deployment logs: `/var/log/kolla/ansible.log`
2. Validation report: `/home/ansible/openstack-validation-report.txt`
3. Kolla-Ansible docs: https://docs.openstack.org/kolla-ansible/
