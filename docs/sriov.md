# X-K8S SRIOV Manual 

## Supported SRIOV NICs

The following  NICs were tested with this implementation. However, other SRIOV capable NICs should work as well.  

- Intel® Ethernet Controller X710 Series 4x10G
  - PF driver : > v2.4.6  
  - VF driver: > v3.5.6  

> please refer to Intel download center for installing latest [Intel Ethernet Controller-X710-Series](https://downloadcenter.intel.com/product/82947/Intel-Ethernet-Controller-X710-Series) drivers  

- Intel 82599ES 10 Gigabit Ethernet Controller  
  - PF driver : > v4.4.0-k
  - VF driver:  > v3.2.2-k  
  
> please refer to Intel download center for installing latest [Intel-® 82599ES 10 Gigabit Ethernet](https://ark.intel.com/products/41282/Intel-82599ES-10-Gigabit-Ethernet-Controller) drivers  

- Mellanox ConnectX-4 Lx EN Adapter  
- Mellanox ConnectX-5 Adapter  

> Network card drivers are available as a part of the various linux distributions and upstream.
To download the latest Mellanox NIC drivers, click [here](http://www.mellanox.com/page/software_overview_eth).

## Pre-Configure Before you install X-K8S  

1. Make sure your SRIOV NIC PF is pluged in with cable and Linked-up.  
    You can check your SRIOV PF link status by  

    ```bash=
    ifconfig enp129s0f0 up
    ethtool enp129s0f0
    ```

2. Enable VF for your SRIOV NIC on each node.  

    e.g.  

    ```bash=
    echo 8 > /sys/class/net/enp129s0f0/device/sriov_numvfs
    ```

3. Edit/Create `/etc/pcidp/config.json` on node with SRIOV
    e.g.

    ```json=
    {
        "resourceList":
        [
            {
                "resourceName": "sriov_net_A",
                "rootDevices": ["02:00.0", "02:00.2"],
                "sriovMode": true,
                "deviceType": "netdevice"
            },
            {
                "resourceName": "sriov_net_B",
                "rootDevices": ["02:00.1", "02:00.3"],
                "sriovMode": true,
                "deviceType": "vfio"
            }
        ]
    }
    ```

    `"resourceList"` should contain a list of config objects. Each config object may consist of following fields:

    |     Field      | Required |                    Description                    |                       Type - Accepted values                        |         Example          |
    |----------------|----------|---------------------------------------------------|---------------------------------------------------------------------|--------------------------|
    | "resourceName" | Yes      | Endpoint resource name                            | `string` - must be unique and should not contain special characters | `"sriov_net_A"`          |
    | "rootDevices"  | Yes      | List of PCI address for a resource pool           | A list of `string` - in sysfs pci address format                    | `["02:00.0", "02:00.2"]` |
    | "sriovMode"    | No       | Whether the root devices are SRIOV capable or not | `bool` - true OR false[default]                                     | `true`                   |
    | "deviceType"   | No       | Device driver type                                | `string` - "netdevice"\|"uio"\|"vfio"                               | `"netdevice"`            |

    You can check the PF bus address by `lshw -class network -businfo`  

## Install X-K8S

Before Install X-K8S, Check again following items are performed on each node you desire to use SRIOV.  

- Latest SRIOV NIC driver are installed.  
- SRIOV PF's status are all linked-up.  
- SRIOV VF are all created.
- `/etc/pcidp/config.json` are created and well configured.

## Start Installing X-K8S with SRIOV Support  

1. Edit `extraVars.yml` to enable sriov support.  

   ```yaml=
   ## SRIOV Support
   sriov_enabled : true
   ```

2. Install X-K8S

    ```bash=
    ./x-k8s install
    ```

## Validation  

When installation completes. You use following commands to check if SRIOV-support is successfully enabled.

```bash=
# Should see sriov-device-plugin daemonset is running.
root@node1:~# kubectl get ds -n kube-system
NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                   AGE
kube-flannel                     2         2         2       2            2           beta.kubernetes.io/os=linux     16h
kube-multus-ds-amd64             2         2         2       2            2           beta.kubernetes.io/arch=amd64   16h
kube-proxy                       2         2         2       2            2           beta.kubernetes.io/os=linux     16h
kube-sriov-device-plugin-amd64   2         2         1       2            1           beta.kubernetes.io/arch=amd64   16h
```

And you can check resouce for each node  

```bash=
# You should see sriov with the resouce name you defined in config.json
root@node1:~# kubectl get node node2 -o json | jq .status.capacity
{
  "cmk.intel.com/exclusive-cores": "10",
  "cpu": "36",
  "ephemeral-storage": "164140444Ki",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "intel.com/sriov_net_A": "8",
  "intel.com/sriov_net_B": "8",
  "memory": "65925044Ki",
  "pods": "110"
}

```

## Create Pod with SRIOV VF NIC

1. Create and apply custom network definitation that uses sriov nic from sriov pool.
    e.g.  

    ```yaml=
    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
      name: sriov-net-a
      annotations:
        k8s.v1.cni.cncf.io/resourceName: intel.com/sriov_net_A
    spec:
      config: '{
      "type": "sriov",
      "vlan": 1000,
      "ipam": {
        "type": "host-local",
        "subnet": "10.56.217.0/24",
        "rangeStart": "10.56.217.171",
        "rangeEnd": "10.56.217.181",
        "routes": [{
          "dst": "0.0.0.0/0"
        }],
        "gateway": "10.56.217.1"
      }
    }'
    ```

2. Create and apply the pod.
    e.g.

    ```yaml=
    apiVersion: v1
    kind: Pod
    metadata:
      name: testpod1
      annotations:
        k8s.v1.cni.cncf.io/networks: sriov-net-a
    spec:
      nodeSelector:
        kubernetes.io/hostname: node2
      containers:
      - name: appcntr1
        image: ubuntu
        imagePullPolicy: IfNotPresent
        command: [ "/bin/bash", "-c", "--" ]
        args: [ "while true; do sleep 300000; done;" ]
        resources:
          requests:
            intel.com/sriov_net_A: '1'
          limits:
            intel.com/sriov_net_A: '1'
    ```

3. You should see a nic in pod with the same net CIDR you define in "sriov-net-a"