# 1. Network Architecture

  ![](https://github.com/cjunwchen/installokd311/blob/master/images/network_diagram.png)
  
  System for the installation here
  * Master node: 16G memory/4 vCPU.
  * Compute node: 8G memory/2 vCPU.
  * On all nodes, base OS is RHEL(CentOS 7.5).

# 2. Prepare DNS

It is important to prepare DNS before the installation. The domain used in this example is f5se.io, which is a internal   domain. Here is how DNS record according to the network diagram above, the external DNS used in this installation is Microsoft AD server. 

![](https://github.com/cjunwchen/installokd311/blob/master/images/dnsad.png)

Take note, a wildcard DNS record would need for external to access apps, the DNS record should be point to IP address of node. In this installation, \*.apps.f5se.io is CNAME to app.apps.f5se.io, and app.apps.f5se.io it points to all three node IP.  

# 3. Prepare Centos on master and all other nodes

3.1	Install Centos and update Centos after installation

	#sudo yum update -y
  
3.2	Install docker, and start docker automatically

	#sudo yum install -y docker
	#sudo systemctl start docker && sudo systemctl enable docker

3.3	Install git

	#sudo yum install -y git

3.4	On All Nodes, Create a user for installation to be used in Ansible and also grant root privileges

	[root@okd-master1~]# echo -e 'Defaults:jun !requiretty\njun ALL = (root) NOPASSWD:ALL' | tee /etc/sudoers.d/openshift 
	[root@okd-master1~]# chmod 440 /etc/sudoers.d/openshift 

	if Firewalld is running, allow SSH
	[root@okd-master1 ~]# firewall-cmd --add-service=ssh --permanent 
	[root@okd-master1 ~]# firewall-cmd --reload 
	
3.5	On Master Node, login with a user created above and set SSH keypair with no pass-phrase. This steps is important, otherwise, the ansible installation in later step would fail

	[jun@okd-master1 ~]$ ssh-keygen -q -N "" 
	Enter file in which to save the key (/home/origin/.ssh/id_rsa):

	[jun@okd-master1 ~]$vi ~/.ssh/config
	# create new ( define each node )
	Host okd-master1
	  Hostname okd-master1.f5se.io
	  User jun
	Host okd-node1
	  Hostname okd-node1.f5se.io
	  User jun
	Host okd-node2
	  Hostname okd-node2.f5se.io
	  User jun
	  
	[jun@okd-master1 ~]$chmod 600 ~/.ssh/config
	
	# transfer public-key to other nodes, take note FQDN is used
	[jun@okd-master1~]$ ssh-copy-id okd-master1.f5se.io 
	[jun@okd-master1.f5se.io password: 

	[jun@okd-master1~]$ ssh-copy-id okd-node1.f5se.io 
	[jun@okd-master1~]$ ssh-copy-id okd-node2.f5se.io 

# 5. On Master Node, install openshift

5.1	Install Ansible

	sudo yum install -y epel-release
	sudo yum install -y ansible

5.2	Disable epel-release
	
	sudo vi /etc/yum.repos.d/epel.repo
	# Change the value enabled=1 to 0 (zero).

5.3	Prepare auth
	
	sudo mkdir -p /etc/origin/master/
	sudo touch /etc/origin/master/htpasswd

5.4	Clone openshift-ansible git 
	
	git clone -b release-3.11 https://github.com/openshift/openshift-ansible.git $HOME/openshift-ansible
	
5.5	Create ansible inventory file, here is the [inventory.ini](https://github.com/cjunwchen/installokd311/blob/master/inventory.ini) that used for this example.
	
5.6	Install openshift via ansible

	ansible-playbook -i ./inventory.ini $HOME/openshift-ansible/playbooks/prerequisites.yml
	ansible-playbook -i ./inventory.ini $HOME/openshift-ansible/playbooks/deploy_cluster.yml

5.7	Enable oc bash completion 

	sudo oc completion bash >>/etc/bash_completion.d/oc_completion

5.8	Add user to OpenShift user

	sudo htpasswd -b /etc/origin/master/htpasswd jun [password]

5.9	Add user 'jun' as cluster admin

	oc adm policy add-cluster-role-to-user cluster-admin jun
	
# 6. Deploy Nginx OpenShift router to all node

6.1 Build latest nginx router image

	$ git clone https://github.com/nginxinc/nginx-openshift-router
	$ cd nginx-openshift-router/src/nginx
	
	If you would like to change some default config in nginx, modify the file nginx-config.template file, which located in directoory
	nginx-openshift-router/src/nginx/conf.
	
	$ docker build -t nginx-openshift-router:latest .
 
6.2 Push nginx image to local registry

	$ oc get svc -n default | grep registry
	
	docker-registry    ClusterIP   172.30.82.196   <none>        5000/TCP                  14d
	registry-console   ClusterIP   172.30.219.2    <none>        9000/TCP                  14d
	
	In the exmple above, the first one is local docker registry, the internal FQDN for it would be docker-registry.default.svc
	
	$ docker login -p $(oc whoami -t) -e unused -u unused docker-registry.default.svc:5000
	$ docker tag nginx-openshift-router:latest docker-registry.default.svc:5000/openshift/nginx-openshift-router:latest
	$ docker push docker-registry.default.svc:5000/openshift/nginx-openshift-router:latest

6.3 Delete the default router in OpenShift

	$ oc project default
	$ oc get -o yaml service/router dc/router clusterrolebinding.authorization/router-router-role serviceaccount/router > default-router-node2-backup.yaml
	$ oc delete -f default-router-backup.yaml

6.4 Deploy nginx router

	$ oc adm router router-master1 --images=docker-registry.default.svc:5000/openshift/nginx-openshift-router:0.2 --type='' --selector='kubernetes.io/hostname=okd-master1.f5se.io'
	$ oc adm router router-node1 --images=docker-registry.default.svc:5000/openshift/nginx-openshift-router:0.2 --type='' --selector='kubernetes.io/hostname=okd-node1.f5se.io'
	$ oc adm router router-node2 --images=docker-registry.default.svc:5000/openshift/nginx-openshift-router:0.2 --type='' --selector='kubernetes.io/hostname=okd-node2.f5se.io'
	
6.5 Verify router deployment

	$ oc get pod -o wide
	
	NAME                       READY     STATUS    RESTARTS   AGE       IP            NODE                  NOMINATED NODE
	docker-registry-1-7qgzv    1/1       Running   1          14d       10.128.0.56   okd-master1.f5se.io   <none>
	registry-console-1-cgxrr   1/1       Running   1          14d       10.128.0.66   okd-master1.f5se.io   <none>
	router-master1-1-2pdjc     1/1       Running   0          1h        10.69.30.1    okd-master1.f5se.io   <none>
	router-node1-1-zn99d       1/1       Running   0          1h        10.69.30.10   okd-node1.f5se.io     <none>
	router-node2-1-kdxh2       1/1       Running   0          1h        10.69.30.20   okd-node2.f5se.io     <none>


	
	
	
 





  


