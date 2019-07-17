# X-K8S - Deploy a Kubernetes Cluster for 5G NFV Cloud

X-K8S leverages plugins for the better NFV performance in 5G use senario, like CMK (Cpu Manager for Kubernetes),  
NFD (Node Feature Discovery), intel's userspace-cni-network-plugin, etc.  
And based on the kubespray v2.8.0 as its deploy tool.  

For the detail deploy instruction, check the kubespray's [readme](https://github.com/mJace/x-k8s/blob/develop/kubespray/README.md).  

## Spec

|    Package    |    version    |
|---------------|---------------|
|Kubernetes     |v1.12.3        |
|Docker         |v18.06.1-ce    |
|CMK            |v1.3.1         |
|NFD            |v0.3.0         |
|Multus         |v3.2           |
|Flannel        |v0.10.0        |
|Flannel-CNI    |v0.3.0         |
|SRIOV-CNI      |v1.0.0         |
|SRIOV-Device Plugin |v2.0      | 

## Deploy Node Requirement

1. Python3  
2. pip3  

## Master/Minion Node Requirement

|  Package   |    version          |
|------------|---------------------|
|  Supported Os | Ubuntu 18.04 LTS Server |


## Usage  

### 1. Prepare your Cluster Node.  
1. Disable and delete swap on **all of your nodes**.  
2. If you want to enable sriov support on your kubernetes cluster.  
    a. Create VF on **all of your nodes**.  
    b. Create `/etc/pcidp/config.json` **on each node** base on your SRIOV NIC bus address.  
    You can use `lshw -class network -businfo` to check the bus address of root device.  
    For example, your config.json should look like...    

```
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
Go check [SRIOV manual](https://github.com/mJace/x-k8s/blob/develop/docs/sriov.md) for more information.  
  
3. Enable root account and allow root remote login for **each node**.   
4. Set password free login for root of deploy node **on each node**    
```bash=
# at root of deploy node
ssh-copy-id <node1_ip>
```


### 2. Install x-k8s  
**On Deploy Node**  
1. Install requirement.  

    ```=bash
    cd x-k8s
    sudo pip3 install -r requirements.txt
    ```  

2. Edit hosts.ini in `/x-k8s/kubespray/inventory/mycluster/hosts.ini`  

3. Edit /x-k8s/kubespray/extraVars.yml  
```yaml=

## Helm deployment 
helm_enabled: true

## Multus deployment
kube_network_plugin: flannel
kube_network_plugin_multus: true

## SRIOV Support
## Only set to true when you know what you are doing.
sriov_enabled : false 

## Enable basic auth
# kube_basic_auth: true
## User defined api password
# kube_api_pwd: xk8suser

## Change default NodePort range 
# kube_apiserver_node_port_range: "9000-32767"
```

4. Deploy  

   ```=bash
   su -
   ./x-k8s install
   ```

## CLI  

```=python
X-K8S Installer
Usage:  
    ./x-k8s install [--i=<hosts>]
    ./x-k8s reset [--i=<hosts>]
    ./x-k8s list inventory [--vars]
    ./x-k8s ( -h | --help)
    ./x-k8s ( -v | --version)

Examples:
    ./x-k8s install                     Install x-k8s.
    ./x-k8s install --i kubespray/inventory/custom/hosts.ini
                                        Install x-k8s using custom inventory.
    ./x-k8s reset                       Reset host environment listed in inventory.
    ./x-k8s reset --i kubespray/inventory/custom/hosts.ini
                                        Reset host environment using custom inventory.
    ./x-k8s list inventory              List hosts inventory.
    ./x-k8s list inventory --vars       List hosts inventory with all variables.
    ./x-k8s -h  
    ./x-k8s --help
    ./x-k8s -v
    ./x-k8s --version

Options:
    -h, --help                          Show this message.
    -v, --version                       Show version.
    --vars                              List hosts inventor with all variables.
    --i=<hosts>                         Path to custom inventory hosts.ini
```
