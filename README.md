Install OKD might be challenge sometime if not prepare properly, the document provides steps for the installation. 

[1] Network Architecture

  ![](https://github.com/cjunwchen/installokd311/blob/master/images/network_diagram.png)
  
  There are some System requirements to configure cluster.
  * Master node has up to 16G memory and up to 4 vCPU.
  * Compute node has up to 8G memory and up to 1 vCPU.
  * On all nodes, base OS is RHEL(CentOS) 7.4 or later (this example is based on CentOS 7.5).

[2]	Prepare DNS

It is important to prepare DNS before the installation. The domain used in this example is f5se.io, which is a internal   domain. Here is how DNS record according to the network diagram above, the external DNS used in this installation is Microsoft AD server. 

![](https://github.com/cjunwchen/installokd311/blob/master/images/dnsad.png)

[3] Install Centos

[4] Update Centos

	#sudo yum update -y
  
[5] Install docker, and start docker automatically

	#sudo yum install -y docker
	#sudo systemctl start docker && sudo systemctl enable docker

[6] Install git

	#sudo yum install -y git

[7] On All Nodes, Create a user for installation to be used in Ansible and also grant root privileges

	[root@okd-master1~]# echo -e 'Defaults:jun !requiretty\njun ALL = (root) NOPASSWD:ALL' | tee /etc/sudoers.d/openshift 
	[root@okd-master1~]# chmod 440 /etc/sudoers.d/openshift 

	if Firewalld is running, allow SSH
	[root@okd-master1 ~]# firewall-cmd --add-service=ssh --permanent 
	[root@okd-master1 ~]# firewall-cmd --reload 
	
[8] On Master Node, login with a user created above and set SSH keypair with no pass-phrase. This steps is important, otherwise, the ansible installation in later step would fail

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
	# transfer public-key to other nodes
	[jun@okd-master1~]$ ssh-copy-id okd-master1.f5se.io 
	[jun@okd-master1.f5se.io password: 

	[jun@okd-master1~]$ ssh-copy-id okd-node1.f5se.io 
	[jun@okd-master1~]$ ssh-copy-id okd-node2.f5se.io 

[5]	On Master Node, install openshift

[5.1] Install Ansible

	sudo yum install -y epel-release
	sudo yum install -y ansible

[5.2] Disable epel-release
	
	sudo vi /etc/yum.repos.d/epel.repo
	# Change the value enabled=1 to 0 (zero).

[5.3] Prepare auth
	
	sudo mkdir -p /etc/origin/master/
	sudo touch /etc/origin/master/htpasswd

[5.4] Clone openshift-ansible git 
	
	git clone -b release-3.11 https://github.com/openshift/openshift-ansible.git $HOME/openshift-ansible
	
[5.5] Create ansible inventory file, here is the [inventory.ini](https://github.com/cjunwchen/installokd311/blob/master/inventory.ini) that used for this example.
	
[5.6] Install openshift via ansible

	ansible-playbook -i ./inventory.ini $HOME/openshift-ansible/playbooks/prerequisites.yml
	ansible-playbook -i ./inventory.ini $HOME/openshift-ansible/playbooks/deploy_cluster.yml

[5.7] Enable oc bash completion 

	sudo oc completion bash >>/etc/bash_completion.d/oc_completion

[5.8] Add user to OpenShift user

	sudo htpasswd -b /etc/origin/master/htpasswd jun [password]

[5.9] Add user 'jun' as cluster admin

	oc adm policy add-cluster-role-to-user cluster-admin jun






  


