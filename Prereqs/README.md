# Install Hyperledger Prerequisites

To run the blockchain exercises, several pre-requisites will need to be installed beforehand:

1.	Docker Engine: Version 17.03 or higher
2.	Docker Compose: Version 1.8 or higher
3.	Node: 8.9 or higher (note version 9 is not supported)
4.	npm: v5.x
5.	git: 2.9.x or higher
6.	Python: 2.7.x
7.	GoLang
8.	Hyperledger Composer

If running on Ubuntu, all dependencies listed above can be installed using the following script (link)
```
curl -O https://hyperledger.github.io/composer/latest/prereqs-ubuntu.sh
chmod u+x prereqs-ubuntu.sh
./prereqs-ubuntu.sh
```

If you’re running on Mac OSX, or if you’d prefer to install each of the listed dependencies individually, continue with the following steps.

**1.	Docker**

Docker is a platform that enables developers to build and run applications within isolated containers. In this workshop, each individual blockchain component will run in their own designated container. 

*Mac OS X:*

To install Docker on Mac OS X, please follow the directions at https://docs.docker.com/docker-for-mac/install/#install-and-run-docker-for-mac

*Linux:*

To install Docker on Ubuntu, please follow the directions listed at
https://docs.docker.com/install/linux/docker-ce/ubuntu/

Or run the provided “get-docker.sh” script like so
```
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh				
sudo usermod -aG docker $(USER)
```

**2.	Docker Compose**

*Mac OS X:*

“Compose” is a Docker add-on that simplifies the deployment of multiple containers. Compose should automatically be installed with “Docker for Mac”. 

*Linux:*

If you’re using a Linux system, run the following commands to install compose
```
sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```
**3.	Node.js**

*Mac OS X and Linux:*

To simplfy the management of our Node.js packages and versions, we’ll use “nvm”. This can be installed with the following command
```
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.0/install.sh | bash
```
Once nvm is set up, install node version 8.9.0 with 
```
nvm install 8.90
nvm use 8.9.0
```

**4.	NPM**

Npm is installed along with node.js in the previous step

**5.	Git**

*Mac OS X:*

Install “brew”, which is the package manager for Mac OS X
```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Next, install git with 
```
brew install git
```

*Linux:*

```
sudo apt-get install git -y
```

**6.	Python**

*Mac OS X:*

Python 2.7x is included by default on Mac OS X, so this step can be skipped

*Linux:*

If you’re running Ubuntu, Python can be installed with the following steps
```
sudo apt update -y
sudo apt upgrade -y
sudo apt install python2.7 python-pip -y
```

**7.	GoLang**

*Mac OS X:*

Visit https://golang.org/dl/ and download the latest “.pkg” file, titled go1.10.3.darwin-amd64.pkg. Next, click on the downloaded file and step through the instructions.

*Linux:*

```
# download tarball
wget https://dl.google.com/go/go1.10.3.linux-amd64.tar.gz

# extract archive to /usr/local
tar -C /usr/local -xzf go1.10.3.linux-amd64.tar.gz

# add go directory to global path
export PATH=$PATH:/usr/local/go/bin
```
See https://golang.org/doc/install#tarball for additional information/troubleshooting


**8.	Composer**

*Mac OS X and Linux:*

```
npm install -g composer-cli composer-playground
```

Additional documentation:
https://hyperledger.githurb.io/composer/latest/installing/development-tools


# Running in a Virtual Machine

For your convenience, the prerequisites listed above have also been pre-installed in a Virtual Machine and copied to an image. This allows a simple alternative to the install steps above.

To import the image, first install Virtualbox and Vagrant using the links below.

Virtualbox
https://www.virtualbox.org/wiki/Downloads

Vagrant
https://www.vagrantup.com/docs/providers/installation.html

Next, run the following set of commands. These commands will download the vagrant image and start a VM based off that image.
```
#vagrant init kkbankol/hyperledger --box-version 1.1
wget https://raw.githubusercontent.com/raheelzubairy/Blockchain-Hyperledger-Advocacy/master/Prereqs/vagrant/Vagrantfile
vagrant up
```

At this point, you should be able to ssh into the VM with the following command
```
vagrant ssh
```

If you encounter this error
```
Stderr: VBoxManage: error: RawFile#0 failed to create the raw output file 
```

Please replace your Vagrantfile with the one in the [vagrant](https://github.com/raheelzubairy/Blockchain-Hyperledger-Advocacy/blob/master/Prereqs/vagrant/Vagrantfile) directory in this repository, and then run
`vagrant up` again
