# Guide to Deploying RHOSO with OpenShift and OpenStack

This guide provides step-by-step instructions to set up a RHOSO environment using GitOps and following the RHOSO Hands on Lab. Follow the steps carefully to ensure successful deployment.

---

## Prerequisites - [rhoso-gitops](https://github.com/turbra/rhoso-gitops)

### Bastion Host Requirements
Install the necessary Python packages:

```bash
pip install -r requirements.txt
```

Install the necessary ansible-galaxy collection(s):
```bash
ansible-galaxy install -r requirements.yml
```

---

## Setting Up ArgoCD

1. Install ArgoCD:

```bash
./base/gitops/deployment.playbook
```

2. Grant the ServiceAccount for ArgoCD the ability to manage the cluster - [showroom_osp-on-ocp](https://github.com/turbra/showroom_osp-on-ocp)

```bash
oc adm policy add-cluster-role-to-user cluster-admin -z openshift-gitops-argocd-application-controller -n openshift-gitops
```

3. Extract the password from the admin user Secret:

```bash
argoPass=$(oc get secret/openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
echo $argoPass
```

4. Get the Route for the OpenShift GitOps server:

```bash
argoURL=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}')
echo $argoURL
```

---

## Deploying Prerequisites for RHOSO Environment - [rhoso-gitops](https://github.com/turbra/rhoso-gitops)

1. Deploy the prerequisites for the RHOSO environment:

```bash
oc create --save-config -k applications/base/prerequisites
```

2. Deploy the prerequisites for the RHOSO control plane installation:

```bash
oc create --save-config -k applications/base/openstack-operator
```

3. Deploy the application manifest for networking configuration - [rhoso-gitops](https://github.com/turbra/rhoso-gitops) & [showroom_osp-on-ocp](https://github.com/turbra/showroom_osp-on-ocp)

   - Created `application-network-config.yaml` and `kustomization.yaml` that point to `showroom_osp-on-ocp` repo

```bash
oc create --save-config -k applications/base/network-configuration
```
If your cluster is RHOCP 4.14 or later and it has OVNKubernetes as the network back end, then you must enable global forwarding so that MetalLB can work on a secondary network interface.

1. Check the network back end used by your cluster:
```bash
oc get network.operator cluster --output=jsonpath='{.spec.defaultNetwork.type}'
```
2. If the back end is OVNKubernetes, then run the following command to enable global IP forwarding:
```bash
oc patch network.operator cluster -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"gatewayConfig":{"ipForwarding": "Global"}}}}}' --type=merge
```

3. NodeNetworkConfigurationPolicy (nncp) resource is used to configure RHOSO openstack services network isolation:
```bash
oc get nncp
```
Review the NetworkAttachmentDefinition (nad) resources for each isolated network to attach a service pod to the corresponding network:
```bash
oc get Network-Attachment-Definitions -n openstack
```

4. Review the MetalLB IP address range. You use the MetalLB Operator to expose internal service endpoints on the isolated networks. By default, the public service endpoints are exposed as RHOCP routes.:
```bash
oc get IPAddressPools -n metallb-system
```
5. Review the L2Advertisement resource which will define which node advertises a service to the local network which has been preconfigured for your demo environment:
```bash
oc get L2Advertisements -n metallb-system
```
6. Finally, review the data plane network. A NetConfig custom resource (CR) is used to configure all the subnets for the data plane networks. You must define at least one control plane network for your data plane. You can also define VLAN networks to create network isolation for composable networks, such as InternalAPI, Storage, and External. Each network definition must include the IP address assignment:
```bash
oc get netconfigs -n openstack
oc describe netconfig openstacknetconfig -n openstack
```

---

## Setting Up RHOSO 18 Control Plane - [showroom_osp-on-ocp](https://github.com/turbra/showroom_osp-on-ocp)

### NFS Configuration
In the bastion host:

1. Create NFS shares for cinder and glance:

```bash
mkdir -p /nfs/cinder /nfs/glance && chmod 777 /nfs/cinder /nfs/glance
```

### Creating VM for Dataplane

2. On the hypervisor host:

```bash
sudo -i
cd /var/lib/libvirt/images
cp rhel-9.4-x86_64-kvm.qcow2 rhel9-guest.qcow2
qemu-img info rhel9-guest.qcow2
qemu-img resize rhel9-guest.qcow2 +90G
chown -R qemu:qemu rhel9-*.qcow2
virt-customize -a rhel9-guest.qcow2 --run-command 'growpart /dev/sda 4'
virt-customize -a rhel9-guest.qcow2 --run-command 'xfs_growfs /'
virt-customize -a rhel9-guest.qcow2 --root-password password:redhat
virt-customize -a rhel9-guest.qcow2 --run-command 'systemctl disable cloud-init'
virt-customize -a /var/lib/libvirt/images/rhel9-guest.qcow2 --ssh-inject root:file:/root/.ssh/id_rsa.pub
virt-customize -a /var/lib/libvirt/images/rhel9-guest.qcow2 --selinux-relabel
qemu-img create -f qcow2 -F qcow2 -b /var/lib/libvirt/images/rhel9-guest.qcow2 /var/lib/libvirt/images/osp-compute-0.qcow2
virt-install --virt-type kvm --ram 16384 --vcpus 4 --cpu=host-passthrough --os-variant rhel8.4 --disk path=/var/lib/libvirt/images/osp-compute-0.qcow2,device=disk,bus=virtio,format=qcow2 --network network:ocp4-provisioning --network network:ocp4-net --boot hd,network --noautoconsole --vnc --name osp-compute0 --noreboot
virsh start osp-compute0
```
**One-liner command:**

```bash
cd /var/lib/libvirt/images && cp rhel-9.4-x86_64-kvm.qcow2 rhel9-guest.qcow2 && qemu-img info rhel9-guest.qcow2 && qemu-img resize rhel9-guest.qcow2 +90G && chown -R qemu:qemu rhel9-*.qcow2 && virt-customize -a rhel9-guest.qcow2 --run-command 'growpart /dev/sda 4' && virt-customize -a rhel9-guest.qcow2 --run-command 'xfs_growfs /' && virt-customize -a rhel9-guest.qcow2 --root-password password:redhat && virt-customize -a rhel9-guest.qcow2 --run-command 'systemctl disable cloud-init' && virt-customize -a /var/lib/libvirt/images/rhel9-guest.qcow2 --ssh-inject root:file:/root/.ssh/id_rsa.pub && virt-customize -a /var/lib/libvirt/images/rhel9-guest.qcow2 --selinux-relabel && qemu-img create -f qcow2 -F qcow2 -b /var/lib/libvirt/images/rhel9-guest.qcow2 /var/lib/libvirt/images/osp-compute-0.qcow2 && virt-install --virt-type kvm --ram 16384 --vcpus 4 --cpu=host-passthrough --os-variant rhel8.4 --disk path=/var/lib/libvirt/images/osp-compute-0.qcow2,device=disk,bus=virtio,format=qcow2 --network network:ocp4-provisioning --network network:ocp4-net --boot hd,network --noautoconsole --vnc --name osp-compute0 --noreboot && virsh start osp-compute0
```

### Configuring Ethernet Devices on Compute

3. Configure the Ethernet devices:

```bash
ssh root@192.168.123.61
nmcli con add con-name "static-eth0" ifname eth0 type ethernet ip4 172.22.0.100/24 ipv4.dns "172.22.0.89"
nmcli con up "static-eth0"
nmcli co delete 'Wired connection 1'
nmcli con add con-name "static-eth1" ifname eth1 type ethernet ip4 192.168.123.61/24 ipv4.dns "192.168.123.100" ipv4.gateway "192.168.123.1"
nmcli con up "static-eth1"
nmcli co delete 'Wired connection 2'
logout
```

**One-liner command:**

```bash
nmcli con add con-name "static-eth0" ifname eth0 type ethernet ip4 172.22.0.100/24 ipv4.dns "172.22.0.89" && nmcli con up "static-eth0" && nmcli --wait 10 dev status | grep -q "static-eth0.*connected" && nmcli co delete 'Wired connection 1' && nmcli con add con-name "static-eth1" ifname eth1 type ethernet ip4 192.168.123.61/24 ipv4.dns "192.168.123.100" ipv4.gateway "192.168.123.1" && nmcli con up "static-eth1" && nmcli --wait 10 dev status | grep -q "static-eth1.*connected" && nmcli co delete 'Wired connection 2' && logout
```

4. Set up SSH keys:

```bash
sudo -i
scp /root/.ssh/id_rsa root@192.168.123.100:/root/.ssh/id_rsa_compute
scp /root/.ssh/id_rsa.pub root@192.168.123.100:/root/.ssh/id_rsa_compute.pub
```

5. Connect to the bastion server:

```bash
ssh root@192.168.123.100
```

6. Create a Secret:

```bash
oc create secret generic dataplane-ansible-ssh-private-key-secret --save-config --dry-run=client --from-file=authorized_keys=/root/.ssh/id_rsa_compute.pub --from-file=ssh-privatekey=/root/.ssh/id_rsa_compute --from-file=ssh-publickey=/root/.ssh/id_rsa_compute.pub -n openstack -o yaml | oc apply -f-
```

---

## Deploying RHOSO Using OpenShift GitOps - [rhoso-gitops](https://github.com/turbra/rhoso-gitops) & [showroom_osp-on-ocp](https://github.com/turbra/showroom_osp-on-ocp)

- Created `application-openstack-ctl-plane.yaml` and `kustomization.yaml` that point to `showroom_osp-on-ocp` repo.
- Created `application-openstack-data-plane.yaml` and `kustomization.yaml` that point to `showroom_osp-on-ocp` repo.

1. Install the control plane:

```bash
oc create --save-config -k applications/base/openstack-ctl-plane
```

2. Create secret for the subcription manager credentials
```bash
echo -n "your_username" | base64
echo -n "your_password" | base64
```

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: subscription-manager
  namespace: openstack
data:
  username: <base64 encoded subscription-manager username>
  password: <base64 encoded subscription-manager password>
EOF
```
3. Install the data plane:

```bash
oc create --save-config -k applications/base/openstack-data-plane
```

---
---

## Accessing OpenStack

1. From the bastion server, access the Control Plane:

```bash
oc rsh -n openstack openstackclient
```

2. Verify OpenStack services:

```bash
cd /home/cloud-admin
openstack compute service list
```

3. Verify OpenStack networks:

```bash
openstack network agent list
```

4. Map the Compute nodes to the Compute cell that they are connected to:
```bash
oc rsh nova-cell0-conductor-0 nova-manage cell_v2 discover_hosts --verbose
```

5. Access to the openstackclient pod
```bash
oc rsh -n openstack openstackclient
```

6. Create image and flavors
```bash
export GATEWAY=192.168.123.1
export PUBLIC_NETWORK_CIDR=192.168.123.1/24
export PRIVATE_NETWORK_CIDR=192.168.100.0/24
export PUBLIC_NET_START=192.168.123.91
export PUBLIC_NET_END=192.168.123.99
export DNS_SERVER=8.8.8.8
openstack flavor create --ram 512 --disk 1 --vcpu 1 --public tiny
curl -O -L https://github.com/cirros-dev/cirros/releases/download/0.6.2/cirros-0.6.2-x86_64-disk.img
openstack image create cirros --container-format bare --disk-format qcow2 --public --file cirros-0.6.2-x86_64-disk.img
```
**One-liner command:**

```bash
export GATEWAY=192.168.123.1 && export PUBLIC_NETWORK_CIDR=192.168.123.1/24 && export PRIVATE_NETWORK_CIDR=192.168.100.0/24 && export PUBLIC_NET_START=192.168.123.91 && export PUBLIC_NET_END=192.168.123.99 && export DNS_SERVER=8.8.8.8 && openstack flavor create --ram 512 --disk 1 --vcpu 1 --public tiny && curl -O -L https://github.com/cirros-dev/cirros/releases/download/0.6.2/cirros-0.6.2-x86_64-disk.img && openstack image create cirros --container-format bare --disk-format qcow2 --public --file cirros-0.6.2-x86_64-disk.img
```

7. Generate a keypair:
```bash
ssh-keygen -m PEM -t rsa -b 2048 -f ~/.ssh/id_rsa_pem
```

8. Create Network and Security for the VM
```bash
openstack keypair create --public-key ~/.ssh/id_rsa_pem.pub default
openstack security group create basic
openstack security group rule create basic --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0
openstack security group rule create --protocol icmp basic
openstack security group rule create --protocol udp --dst-port 53:53 basic
openstack network create --external --provider-physical-network datacentre --provider-network-type flat public
openstack network create --internal private
openstack subnet create public-net \
--subnet-range $PUBLIC_NETWORK_CIDR \
--no-dhcp \
--gateway $GATEWAY \
--allocation-pool start=$PUBLIC_NET_START,end=$PUBLIC_NET_END \
--network public
openstack subnet create private-net \
--subnet-range $PRIVATE_NETWORK_CIDR \
--network private
openstack router create vrouter
openstack router set vrouter --external-gateway public
openstack router add subnet vrouter private-net
```
**One-liner command:**

```bash
openstack keypair create --public-key ~/.ssh/id_rsa_pem.pub default && openstack security group create basic && openstack security group rule create basic --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0 && openstack security group rule create --protocol icmp basic && openstack security group rule create --protocol udp --dst-port 53:53 basic && openstack network create --external --provider-physical-network datacentre --provider-network-type flat public && openstack network create --internal private && openstack subnet create public-net --subnet-range $PUBLIC_NETWORK_CIDR --no-dhcp --gateway $GATEWAY --allocation-pool start=$PUBLIC_NET_START,end=$PUBLIC_NET_END --network public && openstack subnet create private-net --subnet-range $PRIVATE_NETWORK_CIDR --network private && openstack router create vrouter && openstack router set vrouter --external-gateway public && openstack router add subnet vrouter private-net
```

9. Create the Server and a Floating IP
```bash
openstack server create \
    --flavor tiny --key-name default --network private --security-group basic \
    --image cirros test-server
openstack floating ip create public
```

10. Add the floating IP above to the new VM in the next step.
```bash
openstack server add floating ip test-server $(openstack floating ip list -c "Floating IP Address" -f value)
exit
```

11. From the bastion access to the VM.
```bash
ssh cirros@<FLOATING_IP> (password is gocubsgo)

exit
```

## Optional: Enable Horizon
1. From the Bastion:
```bash
oc patch openstackcontrolplanes/openstack-galera-network-isolation -p='[{"op": "replace", "path": "/spec/horizon/enabled", "value": true}]' --type json
oc patch openstackcontrolplane/openstack-galera-network-isolation -p '{"spec": {"horizon": {"template": {"customServiceConfig": "USE_X_FORWARDED_HOST = False" }}}}' --type=merge
```
2. Check that the horizon pods are running after enabling it:
```bash
oc get pods -n openstack
```

3. Get the Route:
```bash
ROUTE=$(oc get routes horizon  -o go-template='https://{{range .status.ingress}}{{.host}}{{end}}')
echo $ROUTE
```
Click the url and log in as username admin password openstack

## Scale out your deployment with a Metal3/Baremetal Cluster Operator provisioned node
TBD
