+++
title = "Terraform & Ansible"
description = "terraform and ansible, one after other"
date = 2023-04-12
draft = false

[taxonomies]
tags = ["terraform", "ansible", "aws"]
categories = ["Automation"]

[extra]
toc = true
comment = true
+++



&emsp; &emsp;In this blog post, we will explore how DevOps teams can leverage IAC tools to simplify their infrastructure management and deployment processes using terraform and then ansible one after other. 
I'm going to make a server on AWS using Terraform and configure server, install necessary packages  using ansible for web page deployment which will display `Hello World!` on its port 80.

## Terraform

![](/images/2023-04-18-21-27-20.png)
&emsp; &emsp; __Terraform__ is an open-source tool for managing infrastructure resources as code, allowing developers to define and manage their infrastructure declaratively using a configuration file. It supports multiple cloud providers and on-premises infrastructure and provides a consistent and repeatable way to provision and manage resources. Prior to running the code and commands below you should have an aws account(free-tier works too), terraform and ansible installed. Make an IAM user and set [aws_cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) on your local machine to use that user's IAM credentials. AWS uses the concept of Identity Access Management(IAM) to manage resource allocation to different users/groups and this allows us to use different IAM setup for different projects. 


`cat main.tf`    #make this file in any directory you want
```bash
#making resources in Singapore
provider "aws" {      
  region  = "ap-southeast-1"
}

#supply your pc public key, type ssh-keygen and press enter multiple times to create new
resource "aws_key_pair" "key_pair" {
  key_name = "mynewkeypair"
  public_key = "${file("/home/pawan/.ssh/id_rsa.pub")}"  
  
}

#making a free-tier ubuntu virtual machine
resource "aws_instance" "web_server" {     
  ami           = "ami-0a72af05d27b49ccb"
  instance_type = "t2.micro"
  key_name      = "mynewkeypair"  

  tags = {
    Name = "WebServerInstance"
  }

  vpc_security_group_ids = [aws_security_group.web_server_sg.id]
}

#adding firewall/security_group rules for ssh,http and outgoing traffic
resource "aws_security_group" "web_server_sg" {    
  name_prefix = "web_server_sg"

  ingress {
    from_port = 22
    to_port   = 22
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 80
    to_port   = 80
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

#data blocks invokes the previously deployed resources
data "aws_instance" "web_server_data" {
  instance_id = aws_instance.web_server.id
}

#displaying the public ip of the server on terminal
output "public_ip" {   
  value = data.aws_instance.web_server_data.public_ip
}
```

It's essential that you execute the commands below from the same directory where the above file is at.

`terraform init`  #checks all the configuration files inside the current working directory and downloads the plugins required according to the resources defined

     
![](/images/2023-04-12-14-52-03.png)

`terraform plan` #shows what is going to be provisioned in details if applied

![](/images/2023-04-12-14-55-54.png)

`terraform apply` #when we do `terraform apply` it prompts us for conformation, we should reply it with `yes` to proceed to provision the server on aws .

![](/images/2023-04-12-15-21-43.png)

`public_ip` variable is set above to display the public ip of the server, which ansible will use  to ssh into the machine and run the ansible_playbook which we'll write below.

We can check the aws console on the browser to confirm the above terraform configuration.

![](/images/2023-04-12-15-28-42.png)

The `Name` supplied in the above `main.tf` matches with the configured server and the public_ip is same as displayed during `terraform apply`. Make note of that ip, since we'll be using this ip to provision web server. Do ssh into this new machine and add the id_rsa.pub file into the new server.


`ssh ubuntu@54.255.87.157`  #ssh should work , which is the necessay condition to run the below ansible playbook

## Ansible

&emsp; &emsp; __Ansible__ is an open-source automation tool that allows developers to automate the deployment and management of infrastructure resources. It uses YAML to define infrastructure as code and supports multiple operating systems and cloud providers. Ansible is agentless, making it easy to manage and scale infrastructure environments. Ansible uses a main file considered as playbook which contains instructions to run on remote servers.

`cat aws_playbook.yml`  #make a file called aws_playbook and paste this content
```yaml
- name: Getting new servers IP address
  hosts: localhost
  gather_facts: no
  vars_prompt:         #prompts to ask server public ip
    - name: target_host
      prompt: Please provide the IP of the new server
      private: no
  tasks:
    - add_host: name="{{ target_host }}" groups=dynamically_created_hosts

- name: Preapring the new server
  remote_user: ubuntu
  hosts: dynamically_created_hosts   
  gather_facts: no
  become: yes
  become_user: root
  become_method: sudo
  vars:
    ansible_ssh_port: 22
  tasks:
    - name: Update and Upgrade apt packages to the latest available
      apt:
        update_cache: yes
        cache_valid_time: 3600
        upgrade: yes

    - name: Installing necessary packages 
      apt: name={{ item }} state=present
      with_items:
        - vim
        - tmux
        - nginx

    - name: Enable ufw firewall
      ufw:
        state: enabled
        rule: allow
        proto: tcp
        port: "{{ item }}"
      with_items:
        - 22
        - 80

    - name: Create "Hello World" HTML file 
      copy:
        content: "<h1>Hello World</h1>"
        dest: "/var/www/html/index.html"
        mode: "0644"

    - name: Start Nginx service
      service:
        name: nginx
        state: started
        enabled: true
```
`ansible-playbook  aws_playbook.yml`   #runs the above file, which asks for the public_ip of the server we just provisioned above.
![](/images/2023-04-12-16-55-20.png)
&emsp; &emsp;The yellow color lines are the changes made on the server and green ones are not changed, since those configurations exist already. Ansible follows the idempotency philoshopy, thus no matter how many times you run this file, only the new changes are going to be applied, the changes which align with the configuration files remain unchanged.

![](/images/2023-04-12-17-03-32.png)
Btw, our web server is up.


`terraform destroy`   #this deletes the sever from the aws, it prompts us and we have to supply `yes` to delete the server
![](/images/2023-04-12-17-05-32.png)

If we refresh the aws console, it says that web server is terminated, which gets removed after a while.![](/images/2023-04-12-17-06-37.png)

&emsp; &emsp;You can add more stuffs to both the terraform and ansible files above to do much more. 

**Images above are scraped through kodekloud,google except screenshots**
