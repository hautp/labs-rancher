# Labs Rancher

## 1. Create droplets and run Ansible bootstrap
### Topology
> Note: In this lab, I used `CentOS` like a main OS to install Rancher and Kubernetes.
Image!

### Create droplets and load bootstrap

Using `Vagrant` to create VMs from `Vagrantfile` and `Ansible` will load **initialize_new_droplet.yml** playbook to install necessary software (included **update packages** and install **repository**, **ntp**, **docker**).

```bash
vagrant up
```

The output example:
```
123
```

You have to wait for a moments to `Vagrant` create VMs and load Ansible bootstrap. 


### Verify droplets status 
After `Vagrant` ran provisioning, you could verify with below command:

```bash
vagrant status
```

The output example:
```
123
```

## 2. Deploy Rancher HA on single node
### Topology
Image!

### Deploy and run Rancher on nodes using Docker

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

With:
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

With:
- **<YOUR.DNS.NAME>** - Your domain address

### Config HA for Rancher by using Nginx (or HAProxy)

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
