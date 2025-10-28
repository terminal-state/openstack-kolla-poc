# OpenStack 2025.1 (Epoxy) Automated Deployment

Complete automation for deploying OpenStack 2025.1 using Kolla-Ansible on KVM virtual machines.

## Architecture

- **Control Node (ctrl-01)**: 192.168.100.100
  - 16 vCPU, 64GB RAM, 200GB system + 500GB data storage
  - All OpenStack control plane services
  - MariaDB, RabbitMQ, Memcached, HAProxy
  - Keystone, Nova, Neutron, Glance, Cinder, Horizon, Placement

- **Compute Nodes (cmp-01, cmp-02, cmp-03)**: 192.168.100.101-103
  - 8 vCPU, 32GB RAM, 100GB storage each
  - Nova-compute, Nova-libvirt
  - Neutron OpenvSwitch agents

- **Networks**:
  - Management: 192.168.100.0/24 (API traffic, SSH)
  - Storage: 192.168.101.0/24 (Cinder, shared storage)
  - Virtual IP (VIP): 192.168.100.200 (HAProxy frontend)

## Prerequisites

- Rocky Linux 9 or AlmaLinux 9 as KVM host
- Minimum resources:
  - CPU: 48+ cores (16 + 8×3 + overhead)
  - RAM: 128GB+ (64 + 32×3 + overhead)
  - Storage: 1TB+
- Network bridge `br0` configured on host
- Internet connectivity

## Quick Start

### Complete Deployment

Clone and deploy:
```bash
cd /etc/ansible
git clone https://github.com/terminal-state/openstack-kolla-poc.git openstack-kolla
cd openstack-kolla

# Run full deployment (skip prechecks due to known false positives)
ansible-playbook -i ../inventory/hosts.ini site.yml --skip-tags prechecks
```

**Note**: The deployment skips Kolla prechecks because they produce false positives for libvirt socket files on compute nodes. The host libvirt is properly stopped, but socket files remain, causing harmless warnings.

Validate deployment:
```bash
ansible-playbook -i ../inventory/hosts.ini site.yml --tags init
```

**Total deployment time**: ~30-45 minutes

### Using tmux for Long-Running Deployments
```bash
# Start new tmux session
tmux new -s openstack

# Run deployment
ansible-playbook -i ../inventory/hosts.ini site.yml --skip-tags prechecks

# Detach from session: Ctrl+b, then d
# Reattach later: tmux attach -s openstack
```

## Post-Deployment Access

### Horizon Dashboard

- **URL**: http://192.168.100.200
- **Admin Username**: `admin`
- **Admin Password**: Found in `/etc/kolla/passwords.yml` on ctrl-01 (keystone_admin_password)

### Command Line Access

SSH to control node:
```bash
ssh -i /home/ansible/.ssh/id_ed25519 ansible@192.168.100.100
```

Source admin credentials:
```bash
source /home/ansible/kolla-venv/bin/activate
source /etc/kolla/admin-openrc.sh
```

Verify deployment:
```bash
openstack compute service list
openstack network agent list
openstack service list
```

Expected output:
- 3 nova-compute services (one per compute node)
- 7 network agents (4 on ctrl-01, 3 OVS agents on compute nodes)
- 7 OpenStack services (keystone, nova, neutron, glance, cinder, placement, cinderv3)

## Deployment Stages

The automation runs through these stages:

1. **KVM Host Setup** (~5 min)
   - Install virtualization packages
   - Configure nested virtualization
   - Setup NTP server
   - Generate SSH keys

2. **VM Creation** (~10 min)
   - Create libvirt networks and storage pools
   - Download Rocky Linux 9 cloud image
   - Create and boot VMs with cloud-init

3. **VM Prerequisites** (~15 min)
   - Wait for cloud-init completion
   - Install Docker
   - Configure networking (management + storage networks)
   - Setup NTP clients
   - Disable firewalld (PoC only)
   - Stop host libvirt services

4. **Kolla Bootstrap** (~10 min)
   - Install Kolla-Ansible in Python venv
   - Generate Kolla passwords
   - Deploy Kolla configuration
   - Create Kolla inventory

5. **OpenStack Deployment** (~30-40 min)
   - Bootstrap servers (Docker setup, volume creation)
   - Deploy all OpenStack services
   - Wait for container stabilization
   - Generate admin credentials file

6. **Validation** (~2 min)
   - Verify all services are operational
   - Confirm compute nodes registered
   - Validate network agents

## Manual Deployment Steps

Run individual phases if needed:

### Deploy VMs Only
```bash
ansible-playbook -i ../inventory/hosts.ini deploy_vms.yml
```

### Deploy OpenStack Only (VMs must exist)
```bash
ansible-playbook -i ../inventory/hosts.ini deploy_openstack.yml --tags bootstrap,deploy
```

### Validate Existing Deployment
```bash
ansible-playbook -i ../inventory/hosts.ini site.yml --tags init
```

## Clean Rebuild

Completely destroy and rebuild the environment:
```bash
# Destroy all VMs
for vm in ctrl-01 cmp-01 cmp-02 cmp-03; do
  virsh destroy $vm 2>/dev/null || true
  virsh undefine $vm --remove-all-storage 2>/dev/null || true
done

# Remove VM disk images
rm -f /var/lib/libvirt/images/ctrl-01-* /var/lib/libvirt/images/cmp-0*

# Verify cleanup
virsh list --all
ls /var/lib/libvirt/images/

# Redeploy from scratch
cd /etc/ansible/openstack-kolla
ansible-playbook -i ../inventory/hosts.ini site.yml --skip-tags prechecks
```

## Project Structure
```
openstack-kolla/
├── site.yml                      # Main orchestration playbook
├── deploy_vms.yml               # VM creation and configuration
├── deploy_openstack.yml         # OpenStack deployment with Kolla
├── validate_openstack.yml       # Deployment validation
├── roles/
│   ├── 00_kvm_host/            # KVM host preparation
│   ├── 01_vm_factory/          # VM creation with cloud-init
│   ├── 02_vm_prerequisites/    # OS configuration (Docker, networking)
│   ├── 03_kolla_bootstrap/     # Kolla-Ansible setup
│   ├── 04_kolla_deploy/        # OpenStack service deployment
│   └── 05_openstack_init/      # Service validation
└── files/
    └── kolla/                   # Kolla configuration templates
        ├── globals.yml.j2
        └── passwords.yml.j2
```

## Configuration

### Key Configuration Files

- **Kolla Globals**: `files/kolla/globals.yml.j2`
  - Network interfaces
  - Enabled services
  - Docker registry settings

- **Inventory**: `/etc/ansible/inventory/hosts.ini`
  - VM hostnames and IPs
  - Group assignments

- **Group Variables**: 
  - `group_vars/all.yml` - Global settings
  - `group_vars/control.yml` - Control node specs
  - `group_vars/compute.yml` - Compute node specs

### Customization

Change VM specifications:
```bash
# Edit compute node resources
vim group_vars/compute.yml

# Edit control node resources
vim group_vars/control.yml
```

Enable additional OpenStack services:
```bash
# Edit Kolla globals template
vim files/kolla/globals.yml.j2

# Set enable_<service>: "yes" for desired services
```

## Troubleshooting

### Check Deployment Status

View Kolla deployment logs:
```bash
ssh ansible@192.168.100.100
sudo tail -f /var/log/kolla/ansible.log
```

Check container status:
```bash
ssh ansible@192.168.100.100
sudo docker ps -a
```

View specific container logs:
```bash
ssh ansible@192.168.100.100
sudo docker logs nova_compute
sudo tail -f /var/lib/docker/volumes/kolla_logs/_data/nova/nova-compute.log
```

### Verify Services
```bash
ssh ansible@192.168.100.100
source /home/ansible/kolla-venv/bin/activate
source /etc/kolla/admin-openrc.sh

# Check compute services (should show 3 compute nodes)
openstack compute service list

# Check network agents (should show 7 agents)
openstack network agent list

# Check all OpenStack services
openstack service list
```

### Common Issues

**Issue**: Prechecks fail with libvirt socket warnings
**Solution**: This is expected. Always use `--skip-tags prechecks` as documented.

**Issue**: Compute nodes not registering
**Solution**: 
1. Check firewalld is disabled: `ssh ansible@192.168.100.101 'sudo systemctl status firewalld'`
2. Verify RabbitMQ connectivity: `ssh ansible@192.168.100.101 'nc -zv 192.168.100.100 5672'`
3. Check nova-compute logs for errors

**Issue**: Containers fail to start with permission errors
**Solution**: Volume permissions are automatically fixed by the automation. If issues persist, manually run:
```bash
ssh ansible@192.168.100.100 'sudo chmod 777 /var/lib/docker/volumes/kolla_logs/_data'
```

**Issue**: Nova registration timeout during deployment
**Solution**: This is a known Kolla-Ansible registration check timeout. The services may still be operational. Verify manually:
```bash
ssh ansible@192.168.100.100 'source /etc/kolla/admin-openrc.sh && openstack compute service list'
```

## Known Limitations

- **Firewall**: Disabled for PoC simplicity. Production deployments should configure proper firewall rules.
- **Single Control Node**: No high availability. Production should use 3+ control nodes.
- **No Ceph**: Uses local storage only. Production should deploy Ceph for shared storage.
- **Nested Virtualization**: Required for compute nodes. Ensure CPU supports it.

## Production Considerations

This is a **PoC environment** for testing and development. Before production:

1. Enable and configure firewalld with proper rules
2. Deploy 3+ control nodes for HA
3. Add Ceph for shared storage
4. Configure SSL/TLS for all services
5. Implement proper backup strategies
6. Configure monitoring (Prometheus, Grafana)
7. Set strong passwords (not auto-generated)
8. Review and harden security settings

## Support and Documentation

- **Project Repository**: https://github.com/terminal-state/openstack-kolla-poc
- **Kolla-Ansible Docs**: https://docs.openstack.org/kolla-ansible/latest/
- **OpenStack Docs**: https://docs.openstack.org/2025.1/

## License

This project is provided as-is for educational and testing purposes.
