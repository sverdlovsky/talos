Homelab
=======

[Talos Linux](https://talos.dev) based kubernetes home cluster. Uses [Cilium](https://cilium.io) as network controller.

Setup
-----

Create image on [Talos Linux Image Factory](https://factory.talos.dev)

-----

### Bare-metal

Burn ISO with:

```sh
dd if=/home/user/Downloads/metal-amd64.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

Boot up with Talos Linux installer flash.


### Virtual Machine

Run each node with:

```sh
qemu-system-x86_64 \
    -enable-kvm \
    -machine type=q35 \
    -cpu host \
    -smp 4 \
    -m 4G \
    -drive file=/dev/sda1,format=raw,if=virtio \
    -cdrom /home/user/Downloads/metal-amd64.iso \
    -boot order=d \
    -netdev bridge,id=net0,br=br0 \
    -device virtio-net-pci,netdev=net0
```

After first reboot remove next lines:

```sh
    -cdrom /home/user/Downloads/metal-amd64.iso \
    -boot order=d \
```

-----

Add to your talos config:

```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

Apply config and bootstrap control plane:

```sh
talosctl apply-config -n 192.168.0.2 --file controlplane.yaml --insecure
talosctl bootstrap -n 192.168.0.2
```

Wait until node has Ready status, then install cilium:

```sh
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/experimental-install.yaml
cilium install \
    --version 1.20.0 \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set gatewayAPI.enabled=true \
    --set gatewayAPI.enableAlpn=true \
    --set gatewayAPI.enableAppProtocol=true \
    --set gatewayAPI.service.externalTrafficPolicy=Local \
    --set routingMode=native \
    --set ipv4.enabled=true \
    --set enableIPv4Masquerade=true \
    --set ipv4NativeRoutingCIDR=192.168.64.0/22 \
    --set ipv6.enabled=true \
    --set enableIPv6Masquerade=false \
    --set ipv6NativeRoutingCIDR=XXXX:XXXX:XXXX:XXXX::/62 \
    --set autoDirectNodeRoutes=true \
    --set bgpControlPlane.enabled=true \
    --set prometheus.enabled=true \
    --set operator.prometheus.enabled=true \
    --set hubble.enabled=true \
    --set hubble.ui.enabled=true \
    --set hubble.relay.enabled=true \
    --set hubble.metrics.enableOpenMetrics=true \
    --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"
```

Install flux:

```sh
flux bootstrap github \
    --owner=sverdlovsky \
    --repository=talos \
    --branch=main \
    --path=flux \
    --personal
```

After BGP configuration:

```sh
cilium upgrade --set enableIPv4Masquerade=false
```

Done.

Source: [Sidero Labs Cilium Guide for Talos](https://docs.siderolabs.com/kubernetes-guides/cni/deploying-cilium#without-kube-proxy-%2B-gateway-api-2)


Cilium connectivity test fix
----------------------------

```sh
kubectl label namespace cilium-test-1 pod-security.kubernetes.io/enforce=privileged
kubectl label namespace cilium-test-ccnp1 pod-security.kubernetes.io/enforce=privileged
kubectl label namespace cilium-test-ccnp2 pod-security.kubernetes.io/enforce=privileged
```

