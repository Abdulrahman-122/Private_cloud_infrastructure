# in order to build a static ip on our vms that are running on kubevirt
- make sure that multus is running (this what will allows us to make another bridge on our vm)
```yml
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml
#wait untill it running on kube-system namespace
kubectl get pods -n kube-system | grep multus
```
- make this file on your host: private-lan.yaml
```yml
nano private-lan.yaml
#put these
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: private-lan
  namespace: pri-cloud
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "bridge",
      "bridge": "br-private",
      "ipam": {}
    }
#apply it
kubectl apply -f private-lan.yaml
#grep multus pods
kubectl get pods -A | grep -i multus
#check crd that you installed 
kubectl get crd network-attachment-definitions.k8s.cni.cncf.io
#check network-attachment-definitions that you applied from the file above
kubectl get network-attachment-definitions

```
- install cni plugin in order to make the bridge
```yml
#on your host do the following
cd /tmp
curl -L -o cni-plugins.tgz https://github.com/containernetworking/plugins/releases/download/v1.5.1/cni-plugins-linux-amd64-v1.5.1.tgz
mkdir -p cni-plugins && tar -xzf cni-plugins.tgz -C cni-plugins
ls -l /tmp/cni-plugins/bridge
#then assign the bridge to every node on your cluster
for n in $(sudo docker ps --format '{{.Names}}' | grep xdr-cluster); do
  sudo docker cp /tmp/cni-plugins/bridge $n:/opt/cni/bin/bridge
  sudo docker exec $n chmod +x /opt/cni/bin/bridge
  echo "== $n"; sudo docker exec $n ls /opt/cni/bin
done
```
- modify each vm to include the following
```yml
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
          - name: default
            bridge: {}
          - name: private          # NEW
            bridge: {}             # NEW
      networks:
      - name: default
        pod: {}
      - name: private              # NEW
        multus:                    # NEW
          networkName: private-lan # NEW
```
- apply each vm again
```yml
k apply -f vm-name.yml (give it probably 3-5 min)
```
- now go inside each vm and do the following
```yml
# stop cloud-init from rewriting the network config
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

# remove the extra file
sudo rm /etc/netplan/60-private.yaml

# one file for both NICs
sudo tee /etc/netplan/50-cloud-init.yaml <<'EOF'
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: true
    enp2s0:
      addresses: [192.168.50.14/24]      #change the last .14 to another number/vm in order to make different ip on this bridge
EOF

sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
ip a show enp2s0
```
- once you did that on all vms now ping them to see whether they can talk to each other or not
```yml
# assume you made: vm1 192.168.50.14,vm2 192.168.50.15
#ping from vm1
ping -c 5 192.168.50.15
#now if you got unreachable or no any response
# okay go the node that contain these vms and make the iptable number from 1 ->0
ip link show br-private
bridge link | grep br-private
sysctl net.bridge.bridge-nf-call-iptables #if this 1 change it to 0
sysctl -w net.bridge.bridge-nf-call-iptables=0
#now test ping again it must work
ping -c 4 192.168.50.15
#1 means -> every packets that goes from 1 vm to another vm -> should be checked by the iprules on the kube-proxy,kindnet workloads
#0 means -> ignore these checks and allow connection
#but: what we did above is temporary once the node restart again -> it will default to 1 so to avoid that
for n in $(sudo docker ps --format '{{.Names}}' | grep xdr-cluster); do
  sudo docker exec $n sh -c 'echo "net.bridge.bridge-nf-call-iptables=0" > /etc/sysctl.d/99-bridge.conf && sysctl --system'
done
