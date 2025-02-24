# NOTE: This repository has been moved to: https://github.com/aldycool/k3s-deploy for more structured approach.

# Ubuntu 20.04 Adaptation README

- Following this: https://digitalis.io/blog/kubernetes/k3s-lightweight-kubernetes-made-ready-for-production-part-1/ with some modifications in this repository to adjust to Ubuntu 20.04 distro.

## All VM Setup
- Prepare 5 VMs: K3S-Control, K3S-01, K3S-02, K3S-03 for Masters, K3S-04 for Workers
  - Control: 2VCPU + 2GB + 20GB
  - Master: 2VCPU + 2GB + 60GB
  - Worker: 2VCPU + 4GB + 60GB
- UserName / Password : k3s / k3s1234
- For hostnames, use hostnames as specified in the Ansible inventory file (`inventory-k3s.yml`), ex: `master01`, `master02`, etc.
- On each VM, we must have two NICs: internal and external. In VMWare, add the NICs in this order: 1. NAT, 2. Bridged. The NAT will be eth0 and will act as internal, and Bridged will be eth1 and will act as external. Because of VMWare (or probably bare-metal) will rename eth0 to ens33 and eth1 to ens37, after hardening later, this will not be compatible. To disable the renaming:
  ```
  # To ensure it is really renamed, check this output
  sudo dmesg | grep -i eth
  # To disable renaming, first edit the the grub
  sudo nano /etc/default/grub
  # Change the following line into this and save:
  GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
  # Save and generate the new grub config with the following line. After it is done, do a reboot
  sudo grub-mkconfig -o /boot/grub/grub.cfg  
  ```
- Prepare the networking:
  ```
  # Check the network interfaces, eth0 (internal) should be listed first, followed by eth1 (external)
  ip a
  # configure netplan
  sudo nano /etc/netplan/00-installer-config.yaml
  # For Control:
  network:
  version: 2
  ethernets:
      eth0:
      addresses: [192.168.232.10/24]
      gateway4: 192.168.232.2
      nameservers:
          addresses: [192.168.232.2]
      eth1:
      dhcp4: true
  # For Master:
  network:
  version: 2
  ethernets:
      eth0:
      addresses: [192.168.232.11/24] # 192.168.232.12 / 192.168.232.13
      gateway4: 192.168.232.2
      nameservers:
          addresses: [192.168.232.2]
      eth1:
      dhcp4: true
  # For Worker:
  network:
  version: 2
  ethernets:
      eth0:
      addresses: [192.168.232.11/41] # 192.168.232.42 / 192.168.232.43
      gateway4: 192.168.232.2
      nameservers:
          addresses: [192.168.232.2]
      eth1:
      dhcp4: true
  # Save and apply
  sudo netplan apply
  ```

## Control VM Setup
- Install Ansible
  ```
  sudo apt update
  sudo apt install software-properties-common
  sudo add-apt-repository --yes --update ppa:ansible/ansible
  sudo apt install ansible
  ansible --version # Ensure the ansible version is greater than 2.9.6
  ```
- Setup SSH Public Keys for Ansible
  ```
  ssh-keygen -t rsa # This will generate ~/.ssh/id_rsa.pub that needs to be copied to all VMs
  ssh-copy-id -i ~/.ssh/id_rsa.pub k3s@192.168.232.11 # and the rest of the VM IP Addresses
  ```
- Install kubectl for later, after the kubeconfig file copied to Control, to set default kubeconfig:
  ```
  export KUBECONFIG=~/k3s-on-prem-production/artifacts/k3s-kube-config
  ```

## Execute Ansible Script
- In Control VM:
  ```
  git clone https://github.com/aldycool/k3s-on-prem-production.git
  cd k3s-on-prem-production
  # Check the `inventory-k3s.yml` to follow the existing networking configuration
  # Now test ansible connection to all VMs
  ansible -m ping -i inventory-k3s.yml all
  # Now execute the main playbook
  ansible-playbook -i inventory-k3s.yml cluster.yml -K
  ```

## Notes on cluster.yml
- The original hardening role is replaced with cis-ubuntu-2004-ansible, taken from: https://github.com/alivx/CIS-Ubuntu-20.04-Ansible, because it is too much difference to work for Ubuntu 20.04.
- Find the "# MOD:" remark all over the code to find my changes.

## Notes Deployments
- Due to PodSecurityPolicy, all pod deployments (except the ones deployed into `kube-system` namespace) have to specify `runAsUser` (which means not running as root), for example:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ...
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ...
  template:
    metadata:
      labels:
        app: ...
    spec:
      securityContext:
        runAsUser: 1000 # Have to indicate this flag
```

## Misc Notes
- To delete / remove completed pods:
  ```
  kubectl delete pod --field-selector=status.phase==Succeeded -A
  ```
- To safely shutdown node:
  ```
  kubectl get node # get all node names
  kubectl drain master01
  # Or use special flags when notified in the error message:
  kubectl drain worker02 --delete-emptydir-data --ignore-daemonsets
  # After all nodes are drained, the nodes can be safely powered off
  # After powering up again, use uncordon to mark all nodes schedulable:
  kubectl uncordon master01
  ```
- To edit falco sidekick secrets (updating its configuration):
  ```
  kubectl get secret -n falco # get the secret name
  kubectl edit secret falcosidekick -n falco # editor will open, put in the base64 encode values here
  ```
- To restart falco pods (editing configuration requires pods to be restarted):
  ```
  kubectl -n falco rollout restart deploy
  # The kubeles delete-pod function must be edited because it was deployed using templates, and the configuration must be empty when the template was written, so an edit to the function and then restart also needed.
  kubectl -n kubeless rollout restart deploy # this also needs to be restarted (delete-pod notification)
  ```
- We still have problem in the new nginx ingress IngressClass resource (https://kubernetes.github.io/ingress-nginx/) so we are still pegged on the old k3s version v1.20.5+k3s1 with the old nginx ingress version 0.45.0. The problem is due to our current setup which uses two nginx controllers for external and internal added with ValidatingWebHookConfiguration, which leads to this problem: https://github.com/kubernetes/ingress-nginx/issues/7546 (still haven't resolved). For now, switch back to old versions.

## Local Temporary Commands
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws revertToSnapshot "C:\Users\Administrator\Documents\Virtual Machines\K3S-Control\K3S-Control.vmx" "Dependencies"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws start "C:\Users\Administrator\Documents\Virtual Machines\K3S-Control\K3S-Control.vmx"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws revertToSnapshot "C:\Users\Administrator\Documents\Virtual Machines\K3S-01\K3S-01.vmx" "Dependencies"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws start "C:\Users\Administrator\Documents\Virtual Machines\K3S-01\K3S-01.vmx"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws revertToSnapshot "C:\Users\Administrator\Documents\Virtual Machines\K3S-02\K3S-02.vmx" "Dependencies"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws start "C:\Users\Administrator\Documents\Virtual Machines\K3S-02\K3S-02.vmx"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws revertToSnapshot "C:\Users\Administrator\Documents\Virtual Machines\K3S-03\K3S-03.vmx" "Dependencies"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws start "C:\Users\Administrator\Documents\Virtual Machines\K3S-03\K3S-03.vmx"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws revertToSnapshot "C:\Users\Administrator\Documents\Virtual Machines\K3S-04\K3S-04.vmx" "Dependencies"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws start "C:\Users\Administrator\Documents\Virtual Machines\K3S-04\K3S-04.vmx"
"C:\Users\Administrator\Documents\Virtual Machines\K3S-05\K3S-05.vmx" "Dependencies"
"C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws start "C:\Users\Administrator\Documents\Virtual Machines\K3S-05\K3S-05.vmx"
scp -r "C:\Users\Administrator\Downloads\k3s-on-prem-production" k3s@192.168.232.10:~/

