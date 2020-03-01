# Labs Rancher
## Indexes
1. [Create droplets and run Ansible bootstrap](https://github.com/hautp/labs-rancher/tree/dev#1-create-droplets-and-run-ansible-bootstrap)
2. [Deploy Rancher HA on single node](https://github.com/hautp/labs-rancher/tree/dev#2-deploy-rancher-ha-on-single-node)
3. [Create custom K8S cluster from RKE](https://github.com/hautp/labs-rancher/tree/dev#3-create-custom-k8s-cluster-from-rke)
4. [Build and deploy a mini service to k8s cluster](https://github.com/hautp/labs-rancher/tree/dev#3-create-custom-k8s-cluster-from-rke)

## 1. Create droplets and run Ansible bootstrap
### - Topology
> Note: In this lab, I used `CentOS` like a main OS to install Rancher and Kubernetes.

![Topology_Vagrant](imgs/topology_vagrant.png)

### - Create droplets and load bootstrap

Using `Vagrant` to create VMs from `Vagrantfile` and `Ansible` will load **initialize_new_droplet.yml** playbook to install necessary software (included **update packages** and install **repository**, **ntp**, **docker**).

```bash
vagrant up
```

The output example:
```
Bringing machine 'master01' up with 'virtualbox' provider...
Bringing machine 'master02' up with 'virtualbox' provider...
Bringing machine 'master03' up with 'virtualbox' provider...
Bringing machine 'worker01' up with 'virtualbox' provider...
Bringing machine 'worker02' up with 'virtualbox' provider...
==> master01: Importing base box 'centos/7'...
==> master01: Matching MAC address for NAT networking...
==> master01: Setting the name of the VM: Rancher_master01_1583083529736_93604
==> master01: Clearing any previously set network interfaces...
...
==> master01: Running provisioner: ansible...
```

You have to wait for a moments to `Vagrant` create VMs and load Ansible bootstrap. 


### - Verify droplets status 
After `Vagrant` ran provisioning, you could verify with below command:

```bash
vagrant status
```

The output example:
```
Current machine states:

master01                  running (virtualbox)
master02                  running (virtualbox)
master03                  running (virtualbox)
worker01                  running (virtualbox)
worker02                  running (virtualbox)
```

## 2. Deploy Rancher HA on single node
### - Topology
![Topology_Rancher](imgs/topology_rancher_single_node.png) 

### - Deploy and run Rancher on nodes using Docker

Login into CentOS host and run installation command below:

+ **Option A: Default Rancher-generated Seft-signed certificate.**

```bash
docker run -d --restart=unless-stopped \
	-p 80:80 -p 443:443 \
	rancher/rancher:latest
```

+ **Option B: Install Rancher with own certificate.**

```bash
docker run -d --restart=unless-stopped \
	-p 80:80 -p 443:443 \
	-v /<CERT_DIRECTORY>/<FULL_CHAIN.pem>:/etc/rancher/ssl/cert.pem \
	-v /<CERT_DIRECTORY>/<PRIVATE_KEY.pem>:/etc/rancher/ssl/key.pem \
	-v /<CERT_DIRECTORY>/<CA_CERTS.pem>:/etc/rancher/ssl/cacerts.pem \
	rancher/rancher:latest
```

- **<CERT_DIRECTORY>** - The path to the directory containing your certificate files. 
- **<FULL_CHAIN.pem>** - The path to your full certificate chain.
- **<PRIVATE_KEY.pem>** - The path to the private key for you certificate.
- **<CA_CERTS>** - The path to the certificate authoriity's certificate.

+ **Option C: Let's Encrypt Certificate**

You must point A record with the hostname that you want to use the Rancher access to the IP of the machine it is running on.

```bash
docker run -d --restart=unless-stopped \
	-p 80:80 -p 443:443 \
	rancher/rancher:latest \
	--acme-domain <YOUR.DNS.NAME>
```

- **<YOUR.DNS.NAME>** - Your domain address

### - Config HA for Rancher by using Nginx (or HAProxy)

> In this lab, I will use Nginx (docker) to HA for 2 Rancher nodes.

- Create `rancher_ha.conf` on node that will be run nginx docker

```bash
vim /opt/rancher_ha.conf
```

```
upstream ha_rancher_http {
    least_conn;
    server 192.168.57.50:80 max_fails=3 fail_timeout=5s;
    server 192.168.57.51:80 max_fails=3 fail_timeout=5s;
}

upstream ha_rancher_https {
    least_conn;
    server 192.168.57.50:443 max_fails=3 fail_timeout=5s;
    server 192.168.57.51:443 max_fails=3 fail_timeout=5s;
}

 server {
    listen 80;
    server_name rancher.hautp.local;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_pass http://ha_rancher;
    }
}

server {
    listen 443;
    server_name rancher.hautp.local;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_pass http://ha_rancher;
    }
}
```

- Run `nginx` docker container with custom config
```bash
docker run -d --name ha_rancher \
	--restart=unless-stopped \
	-p 80:80 -p 443:443 \
	-v /opt/rancher_ha.conf:/etc/nginx/conf.d/rancher_ha.conf \
	nginx:latest 
```

## 3. Create custom K8S cluster from RKE
### - Topology
![Topology_RKE](imgs/topology_rke_rancher.png)

### - Define topology K8S with `rke` command

- You could create empty file `cluster.yml` from `rke` command
```bash
rke config --empty
```

- Or input value into prompt from `rke` command following below step
```bash
rke config 
```

The output example:
```
[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]: ~/.ssh/id_rsa
[+] Number of Hosts [1]: 5
[+] SSH Address of host (1) [none]: 192.168.57.10
[+] SSH Port of host (1) [22]:
[+] SSH Private Key Path of host (192.168.57.10) [none]:
[-] You have entered empty SSH key path, trying fetch from SSH key parameter
...
[+] Add another addon [no]: no
```

### - Build and verify K8S cluster from RKE 
- Create K8S cluster from `cluster.yml`

```bash
rke up --config cluster.yml
```

You must wait a few moments to `rke` create K8S cluster.

- Verify after RKE build K8S successfully

```bash
export KUBECONFIG=$(pwd)/kube_config_rancher-cluster.yml
```

```bash
kubectl get nodes
```

```
NAME            STATUS   ROLES               AGE     VERSION
192.168.57.10   Ready    controlplane,etcd   11m   v1.17.2
192.168.57.11   Ready    controlplane,etcd   11m   v1.17.2
192.168.57.12   Ready    controlplane,etcd   11m   v1.17.2
192.168.57.20   Ready    worker              11m   v1.17.2
192.168.57.21   Ready    worker              11m   v1.17.2
```

- Check health's pods on K8S cluster
```bash
kubectl get pods --all-namespaces
```

```
NAME                                      READY   STATUS      RESTARTS   AGE
canal-29ppn                               2/2     Running     2          11m
canal-8hpvv                               2/2     Running     2          11m
canal-d82zk                               2/2     Running     2          11m
canal-t5pqq                               2/2     Running     2          11m
canal-t6gk5                               2/2     Running     2          11m
coredns-7c5566588d-b8cv2                  1/1     Running     1          11m
coredns-7c5566588d-fr4j5                  1/1     Running     1          11m
coredns-autoscaler-65bfc8d47d-mgjs9       1/1     Running     1          11m
metrics-server-6b55c64f86-nnlxv           1/1     Running     1          11m
rke-coredns-addon-deploy-job-wp8wg        0/1     Completed   0          11m
rke-ingress-controller-deploy-job-n24z8   0/1     Completed   0          11m
rke-metrics-addon-deploy-job-tp9j7        0/1     Completed   0          11m
rke-network-plugin-deploy-job-wx7bj       0/1     Completed   0          11m
```
