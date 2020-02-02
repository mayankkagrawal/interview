# interview
1. #apt-install apache2   //first we have to install apache webserver.
   #service apache2 restart //restart the service.by default apache2 webserver is running on port 80.

2. i used root user as a administrator and whenever ansible goes controller node to managed node, in background it use ssh and for 
   doing this it requires passowrd. password may be key based and password base but for security purpose always use key based.
   # ssh-keygen
   # ssh-copy-id root@managed_node

# yum install ansible                   //install ansible in system
   # cp /etc/ansible/ansible.cfg .      //copy the maine ansible config file in current directory which is in /workspace 
   # vim ansible.cfg 
          1. inventory=/root/interview/inventory
          2. host_key_checking=false //whenever ansible controller node goes to manage node it will not required password.
    
 2.1 To configure docker 
     i configured docker on localhost for testing.
     	- hosts: localhost
  tasks:
     - name: "installing docker" 
       package:
           name: docker-ce
           state: present
     - name: "Restart docker service"
       service:
           name: docker
           state: restarted
           enabled: yes
     - name: "docker"
       package: 
              name: python2-pip
              state: present
     - pip:
          name: docker-py
     - command: "pip install --upgrade pip"
     - name: "centos:latest images pull from docker hub"
       docker_image:
              name: centos
              tag: latest
              pull: yes
     - name: "load mayank name docker"
       docker_container:
                name: "Mayank"
                image: centos
                state: started
                restart: yes
                interactive: yes
                tty: yes
2.2 	create a inventory in working directory
		# vim inventory
			[kubernetes-master-nodes]
			192.168.56.100
			[kubernetes-worker-nodes]
			192.168.56.105
      # create a enviornment file
	  #	vim env_variable
	  Enter the IP Address of the Kubernetes Master node to the ad_addr variable.
ad_addr: 192.168.56.100
cidr_v: 172.16.0.0/16
packages:
		- kubeadm
		- kubectl
services:
		- docker
	    - kubelet
		- firewalld
ports:
		- "6443/tcp"
		- "10250/tcp"
token_file: join_token
create a playbook to configure master node
- hosts: kubernetes-master-nodes
  vars_files:
  - env_variables
  tasks:
  - name: Pulling images required for setting up a Kubernetes cluster
    shell: kubeadm config images pull

  - name: Resetting kubeadm
    shell: kubeadm reset -f
    register: output

  - name: Initializing Kubernetes cluster
    shell: kubeadm init --apiserver-advertise-address {{ad_addr}} --pod-network-cidr={{cidr_v}}
    register: output

  - name: Storing Logs and Generated token for future purpose.
    local_action: copy content={{ output.stdout }} dest=/home/centos/{{ token_file }}

  - name: Copying required files
    shell: |
     mkdir -p $HOME/.kube
     sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config
  - name: Install Network Add-on
    command: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
create a playbook to configure slave node
- hosts: kubernetes-worker-nodes
  vars_files:
  - env_variables
  tasks:
  - name: Copying token to worker nodes
    copy: src=/home/centos/{{ token_file }} dest=/home/centos/join_token

  - name: Joining worker nodes with kubernetes master
    shell: |
     kubeadm reset -f
     cat /home/centos/join_token | tail -2 > out.sh
     sh out.sh
create a prerequisites playbook
- hosts: all
  vars_files:
  - env_variables
  tasks:
  - name: Disabling Swap on all nodes
    shell: swapoff -a

  - name: Commenting Swap entries in /etc/fstab
    replace:
     path: /etc/fstab
     regexp: '(.*swap*)'
     replace: '#\1'
create a playbook to configure node
- hosts: all
  vars_files:
  - env_variables
  tasks:
  - name: Creating a repository file for Kubernetes
    file:
     path: /etc/yum.repos.d/kubernetes.repo
     state: touch

  - name: Adding repository details in Kubernetes repo file.
    blockinfile:
     path: /etc/yum.repos.d/kubernetes.repo
     block: |
      [kubernetes]
      name=Kubernetes
      baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
      enabled=1
      gpgcheck=1
      repo_gpgcheck=1
      gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
  - name: Installing Docker and firewalld
    shell: |
     yum install firewalld -y
     yum install -y yum-utils device-mapper-persistent-data lvm2
     yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
     yum install docker-ce -y
  - name: Installing required packages
    yum:
     name: "{{ item }}"
     state: present
    with_items: "{{ packages }}"

  - name: Starting and Enabling the required services
    service:
     name: "{{ item }}"
     state: started
     enabled: yes
    with_items: "{{ services }}"

  - name: Allow Network Ports in Firewalld
    firewalld:
     port: "{{ item }}"
     state: enabled
     permanent: yes
     immediate: yes
    with_items: "{{ ports }}"

  - name: Enabling Bridge Firewall Rule
    shell: "echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables"
create a setup master node file
- import_playbook: playbooks/prerequisites.yml
- import_playbook: playbooks/setting_up_nodes.yml
- import_playbook: playbooks/configure_master_node.yml
create a setup for slave node file
- import_playbook: playbooks/configure_worker_nodes.yml

2.3 
- hosts: localhost
  vars_files:
       - env_variabbles
   tasks:
    - name: "turning selinux on"
   		selinux:
             policy: targeted
			       state: enforcing
    - name: "start firewalld"
      service:
             name: firewalld
             state: started
	  - name: "turning firewalld on"
      firewalld:
                name: firewalld
                state: enabled
                port: 8081/tcp
    - name: "increase faildelay time to prevent from bruteforce attack"
		  template:
                src: login.defs.j2    #i am using FAIL_DELAY=10000(10 seconds) keyword at the end of the file in 
                                                              /etc/security/time.conf
                dest: /etc/login.defs
	  - name: "use the pam_faildelay in auth section"
      template
		 	  src: sshd.j2
        dest: /etc/pam.d/sshd
		- name: "to block user after 5 unsuccessful upto 2 minutes"
      command: "authconfig --enablefaillock --faillockargs='deny=5 unlcok_time=120' --update "
	  - name: "create a advance intrusion detection enviornment for /etc/passwd and /etc/shadow file"
      template:
          	src: aide.conf.j2
            dest: /etc/aide.conf
    - name: "block the all users who is trying to ssh to root"
      template: 
               src: access.conf.j2      // in this file i write
 																            +:root:ALL
																            -:ALL:ALL 
               dest: /etc/security/access.conf
     - name: "copy the ssh file"
       template:
          src: sshd.j2
          dest: /etc/pam.d/sshd
               
               
               
               
               
               
               
