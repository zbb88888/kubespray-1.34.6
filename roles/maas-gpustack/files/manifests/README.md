# Kube-OVN Geneve manifests

本目录保留 Kube-OVN 相关的手动部署说明。当前 Demo 使用普通 Pod 网络，
不使用 hostNetwork-only 方案。

Kube-OVN 默认使用 Geneve：

```text
UDP 6081
```

外层 Calico 的 workload 封装保护规则针对 IPIP 和 VXLAN，不针对 Geneve。
仍需确认平台安全组和 KubeVirt 网络允许三台 VM 之间双向 UDP 6081。

Kube-OVN 不随 Kubespray 的 `cluster.yml` 标准 CNI 阶段部署。请先执行：

```bash
cd /root/7/kubespray1-31
ansible-playbook -i inventory/maas-test/inventory.ini \
  --become --become-user=root kubeovn.yml
```

按照 role 输出，在控制节点执行生成的：

```text
/tmp/kube-ovn-install.sh
```

默认安装脚本位于：

```text
roles/kubeovn/files/install-kube-ovn-115.sh
```

确认 Kube-OVN 已运行后再部署 GPUStack：

```bash
kubectl get pods -n kube-system -o wide | grep -E 'ovn|kube-ovn'
kubectl get subnet
```

这里不维护 CoreDNS hostNetwork manifests，因为 CoreDNS 使用 Kube-OVN 的
普通 Pod 网络。NodeLocalDNS 仍可按 maas-test inventory 的现有配置关闭。
