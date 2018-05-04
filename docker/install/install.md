## Install Docker using the repository
*  SET UP THE REPOSITORY  
	1. Install required packages. yum-utils provides the yum-config-manager utility, and device-mapper-persistent-data and lvm2 are required by the devicemapper storage driver. <br> ```$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2  ```<br/>
  2. Use the following command to set up the stable repository. You always need the stable repository, even if you want to install builds from the edge or test repositories as well. <br> ```$ sudo yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo  ```<br/>
  3. Optional: Enable the edge and test repositories. These repositories are included in the docker.repo file above but are disabled by default. You can enable them alongside the stable repository. <br> ``` $ sudo yum-config-manager --enable docker-ce-edge ``` <br/> <br> ``` $ sudo yum-config-manager --enable docker-ce-test ``` <br/>

- INSTALL DOCKER CE
1. Install the latest version of Docker CE, or go to the next step to install a specific version: <br> ```$ sudo yum install docker-ce ``` <br/>
2. To install a specific version of Docker CE, list the available versions in the repo, then select and install:
	- List and sort the versions available in your repo. This example sorts results by version number, highest to lowest, and is truncated:  <br> ``` $ yum list docker-ce --showduplicates | sort -r ``` 
``` docker-ce.x86_64    18.04.0.ce-3.el7.centos    docker-ce-edge```<br/> 
		- Install a specific version by its fully qualified package name, which is the package name (docker-ce) plus the version string (2nd column) up to the first hyphen, separated by a hyphen (-), for example, docker-ce-18.03.0.ce. <br> ``` $ sudo yum install docker-ce-<VERSION STRING>``` <br/>
3. Start Docker. <br> ``` $ sudo systemctl start docker```<br/>
4. Verify that docker is installed correctly by running the hello-world image. <br>```$ sudo docker run hello-world ``` <br/>
