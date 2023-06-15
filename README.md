# calico-vpp

### Setup Instructions

1. Copy files and edit calico-vpp.xml file to suit your environment:
```
image location:

      <source file='/opt/images/calico-vpp.img'/>

VPP NIC:

    <interface type='network'>
      <mac address='52:54:16:01:16:07'/>   
      <source network='default' bridge='virbr0'/>
      <target dev='vnet1'/>
      <model type='virtio'/>
      <alias name='net1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </interface>
```

2. Virsh define and start VM
```
virsh define calico-vpp.xml
virsh start calico-vpp
```

3. ssh to the VM (might need to console into the VM to determine its mgt IP)
```
ssh cisco@192.168.122.16
cisco123
```

4. check if k8s is running. If k8s is not running reset and reinitialize kubeadm:
```
kubectl get nodes -o wide
sudo kubeadm reset
sudo kubeadm init --pod-network-cidr=10.97.0.0/16
```

5. Start calico-vpp CNI:
```
kubectl apply -f tigera-operator.yaml 
kubectl apply -f calico-vpp-install-default.yaml
kubectl apply -f calico-vpp.yaml 
kubectl get pods -A
```

6. untaint the node to allow a workload container to run on the master
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

7. Test access to VPP command line. Pod name *** calico-vpp-node-ws9xm *** can be found in 'kubectl get pods -A' output:
```
kubectl -n calico-vpp-dataplane exec -it calico-vpp-node-ws9xm  -c vpp -- vppctl   
```

8. set VPP's srv6 source address and add an srv6-te policy:
```
set sr encaps source addr 2001:ffff:420:1016::2
sr policy add bsid 2001::1 next fc00:0:7:11:e003:: encap
sr steer l3 10.2.3.0/24 via bsid 2001::1
```

9. start Alpine container
```
kubectl apply -f alpine.yaml 
```

10. access alpine container and ping
```
kubectl exec -it alpine /bin/sh

ping 10.2.3.1 -i .4
```

11. tcpdump on calico-vpp VM outside NIC to see if packets are encapsulated in SRv6:
```
admin@sonic01:~$ sudo tcpdump -ni Ethernet16
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on Ethernet16, link-type EN10MB (Ethernet), snapshot length 262144 bytes
02:38:35.913458 IP6 2001:ffff:420:1016::2 > fc00:0:7:11:e003::: IP 10.97.157.48 > 10.2.3.1: ICMP echo request, id 22, seq 8, length 64
02:38:36.213757 IP6 2001:ffff:420:1016::2 > fc00:0:7:11:e003::: IP 10.97.157.48 > 10.2.3.1: ICMP echo request, id 22, seq 9, length 64
```