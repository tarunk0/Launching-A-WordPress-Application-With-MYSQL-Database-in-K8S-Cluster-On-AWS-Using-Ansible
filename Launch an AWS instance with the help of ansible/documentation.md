# Agenda:

 - Launch an AWS instance with the help of ansible.
 - Retrieve the public IP which is allocated to the launched instance.
 - With the help of the retrieved Public IP configure the web server in the launched instance.

## What is Ansible?

 Ansible is great tool in doing configuration on an any OS. What we we have to launch OS using Ansible. So this can be possible using Ansible. But ansible is meant for configuration and for provisioning OS we cab use Terraform. Though Ansible can manage Configuration as well as provisioning there i am using ansible to provision ec2 instance and also for configuring webserver inside that ec2 instance. The task description can be cleared using below figure.
 
![image](https://user-images.githubusercontent.com/92631457/188576261-669a9b32-c67c-40bf-a719-61ca9ba4ae0b.png)

![image](https://user-images.githubusercontent.com/92631457/188576297-09958e3b-a296-4ddd-8d15-e610d85243c6.png)

As you can see in the above figure. That we can use our localhost ip address to behave as a managed node and we will use the SDK to lauch ec2 instance on AWS as ansible is built on python language so we will be be using boto. Boto as it a a API so it has a capability to connect with AWS. 

```
    pip3 insatll boto
 ```

So, the first step is to launch the ec2 instance. Lets start:

```
- hosts: "localhost"

  vars_files:

      - secret_key.yml

  tasks:

  - name: "provisioning OS on AWS using Ansible"

    ec2:

       key_name: "openstack"

       instance_type: "t2.micro"

       image: "ami-052c08d70def0ac62"

       wait: yes

       count: 1

       instance_tags:

          Name: aws_ansible_ec2
 
      vpc_subnet_id: "subnet-0f52803afe10747e7"

       assign_public_ip: yes
 
       region: "ap-south-1"

       state: present

       group_id: "sg-09f6ca37ea9000fc8"

       aws_access_key: "{{aws_access_key}}"

       aws_secret_key: "{{aws_secret_key}}"

    register: X


  - debug:
  

     var:  X.instances[0].private_ip
     
  ```
  
  The above code will launch ec2 instance on AWS. The aws_access_key and aws_secret_key are stored in vault. 
  You can create vault and run ansible playbook via below command:
  
  ``` 
     ansible  create  --vault-id  name@prompt   vault_name.yml    --> creating Vault

     ansible-playbook  --vault-id name@prompt  playbook_name.yml  --> running playbook
 ```
 
 After running second command your ansible-playbook will start running:
 
 ![image](https://user-images.githubusercontent.com/92631457/188579882-8b2a12cd-924e-412e-8403-a04f06ba55e5.png)
 
 ![image](https://user-images.githubusercontent.com/92631457/188580092-7b3432d8-e32c-4b56-a8d6-fab810a88de0.png)
 
 Now we have to set the username and password in the newly launched instance.
 
 ```
    #This will create a username and password in the newly launched ec2 instance

    useradd    username

    passwd     password
```
After creating user update in the /etc/sudoers.

![image](https://user-images.githubusercontent.com/92631457/188581113-b23dc3b3-d019-4289-9d2a-4c329465f5de.png)

After updating in the sudoers file now we have to configure ssh file in /etc/ssh/sshd_config. Give the PasswordAuthentication yes in this configuration file.

![image](https://user-images.githubusercontent.com/92631457/188581243-3a527e23-1a0e-4f01-b2dd-69edca4564b3.png)

Now we have to restart the service of ssh configuration to make the changes come into effect.

```
  service sshd restart           #restarts the sshd services.
```

Generate and copy the ssh keys to the Ec2 Instance:

```
    ssh-keygen                                    # will generate the ssh key

    ssh-copy-id  user_name@ip_of_new_instance     # will copy ssh key to new ec2
```

Now we are good to go. Install wget command. Go in the newly created eg /mydb directory and run the below commads.


    This 👇 command will create a ec2.py dynamic inventory file 

    wget   https://raw.githubusercontent.com/ansible/ansible/stable-2.9/contrib/inventory/ec2.py

    This 👇 command will create a ec2.ini dynamic inventory file

    wget   https://raw.githubusercontent.com/ansible/ansible/stable-2.9/contrib/inventory/ec2.ini


    You need to make executable those two above files

    chmod  +x ec2.py 

    chmod  +x ec2.ini

    You need to give AWS_REGION='ap-south-1' 
               AWS_ACCESS_KEY=XXXX
               AWS_SECRET_KEY=XXXX          inside ec2.ini file.

    After that export all these commands.
    export AWS_REGION='ap-south-1'
    export AWS_ACCESS_KEY=XXXX
    export AWS_ACCESS_SECRET_KEY=XXXX

    Now we have to edit inside ec2.py file. This file written in python 2 but we are using python 3 so we need to edit the header.

    #!/usr/bin/python3


You can find all the dynamic inventory files here. 👇

https://github.com/ansible/ansible/blob/stable-2.9/contrib/inventory/ec2.ini

But what is dynamic inventory in ansible ??

As described in Working with dynamic inventory, Ansible can pull inventory information from dynamic sources, including cloud sources, using the supplied inventory plugins. If the source you want is not currently covered by existing plugins, you can create your own as with any other plugin type.

After that you need to give this /mydb path location in /etc/ansible/ansible.cfg file. Now ansible will know that how many ec2 instances are running on AWS. You can check via below commands:

    ansible  ec2  --list-hosts             --> will give IP of hsots available on AWS

    ansible  ec2  -m  ping                  --> will check wether its pinging or not
    
    
In the above command ec2 is a default inventory of all the ec2 instances running on AWS.

![image](https://user-images.githubusercontent.com/92631457/188583622-3de770d2-8156-4478-9c14-307eed4cf67e.png)

As you can see the ansible is now connected to newly launched ec2 instance. The upper red error is of controller node. Though my controller node is also on AWS so dynamic inventory od ansible also fetch the ip of controller node as well as managed node. So we have to configure webserver in managed node from controller node. You need to create a .html eg. index.html so we can copy that file in the managed node at /var/www/html location. Apache webserver will fetch the .html file from /var/ww/html location.

![image](https://user-images.githubusercontent.com/92631457/188584428-3b32607e-b04d-407a-93fa-ca1acdceb301.png)

The below code will configure webserver in managed node.


    - hosts: "ec2"

      tasks:

      - package:

           name: "httpd"

           state: present

      - copy:

           src: "index.html"

           dest: "/var/www/html"

      - service:
           name: "httpd"


           state: restarted
           
 The below command will run the ansible play-book and configure the web server in managed node.
 
 ![image](https://user-images.githubusercontent.com/92631457/188584968-5c05be68-987e-47b4-8f78-30f5cbe7d0ad.png)


As the code Successfully run. then we can check the webpage via. public ip of ec2 instance/index.html

![image](https://user-images.githubusercontent.com/92631457/188585597-8bbd6778-59dd-462a-ba9f-51e94c6861ee.png)

YAY !! We successfully configured the manged node webserver.
 
 




