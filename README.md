Install OKD might be challenge sometime if not prepare properly, the document provides detail steps for the installation. 

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
  


