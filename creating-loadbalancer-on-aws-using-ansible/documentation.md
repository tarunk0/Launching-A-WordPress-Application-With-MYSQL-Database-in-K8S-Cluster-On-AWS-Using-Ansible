# Creating Loadbalancer on AWS using Ansible


What is Load Balancer ?

Load balancing is defined as the methodical and efficient distribution of network or application traffic across multiple servers in a server farm. Each load balancer sits between client devices and backend servers, receiving and then distributing incoming requests to any available server capable of fulfilling them.

![image](https://user-images.githubusercontent.com/92631457/188587701-2bb090a2-f070-406f-9481-a0e25ad94cc9.png)


## STEP: 1
  - Create 5 ec2 Instances. Here I am taking mixture of t2.micro and t3.micro. You can take any instance type. But be sure take t3.micro for Ansible Node, in which you have to Install Ansible.

      1 - - Ansible Node (Controller Node) -- t3.micro

      1 - - LoadBalancer -- t2.micro

      3 - - Webserver -- t2.micro


![image](https://user-images.githubusercontent.com/92631457/188589107-2c6050fa-4cbc-4539-abab-8847d5b3402b.png)

Check the below figure to understand what are we going to do 👇

![image](https://user-images.githubusercontent.com/92631457/188589343-0b0bd88f-977d-4b3b-be80-f3767fe7673f.png)

## STEP: 2
  - Now we have to first create a role inside ControllerNode. Create two roles one for LoadBalancer and another for Webservers.
  - Set the webservers and loadbalancers inside ansible hosts.
 
![image](https://user-images.githubusercontent.com/92631457/188589633-e2ce3877-497f-4d2b-b34b-6f17982f7358.png)


## STEP: 3
  - create a role folder mkdir /etc/ansible/roles all the roles will be inside this roles folder.

``` 
   ansible-galaxy  init  webserver           # role created for webserver

   ansible-galaxy  init  lbserver            # role created for LoadBalancer
```

Also create a directory to manage hosts files. Here i have created:

```
  mkdir     /etc/ansible/roles/projects
```
Here a complete picture of above explanation looks like:

![image](https://user-images.githubusercontent.com/92631457/188590740-65033782-17e8-4fdf-9eda-9017264c5979.png)

Also you need to set the path of role inside ansible configuration file (ansible.cfg).

## STEP: 4

  - Now we can start writing the playbook inside projects directory.

![image](https://user-images.githubusercontent.com/92631457/188590975-3ea7bf84-037f-4e80-a6dd-ce4f097ae71a.png)

    - hosts: web
      roles:
      - role: webserver



    - hosts: LoadBalancer

      roles:
      - role: lbserver
      
 


   - After that we have to configure webserver of each instance. We have write the tasks in tasks folder and handlers in handlers folder.

![image](https://user-images.githubusercontent.com/92631457/188591799-5a983ac6-37d6-4268-ac59-62030e14f4dc.png)

   - Now we can write the tasks inside main.yml
 
 ![image](https://user-images.githubusercontent.com/92631457/188592083-e113f088-80e1-446f-9e1c-0edb0909258f.png)

    - name: "install httpd"
      package:
        name: "httpd"
        state: present
    - name: "copy the content"
      copy:
        src: "/etc/ansible/roles/index.html"
        dest: /var/www/html/
    - name: "restart httpd"
      service:
        name: "httpd"
        state: started
        
Here we have completed with configuring webserver. You can also use Handlers to restart apache server. Now we have to configure the Load Balancer. First install haproxy inside Controller node.

    yum   install   haproxy   -y     #it will install haproxy
    
Then you need to change the port number binding. You can use any port eg 1234. Here i have used 8080. also you need to provide the public ip of all the instances with 80 port. To give Ip randomly here we can use Jinja Template to extract the hostname of each ec2 instance.


![image](https://user-images.githubusercontent.com/92631457/188595085-aacefb6e-d6ac-497d-8fb7-e4e331773eaf.png)

![image](https://user-images.githubusercontent.com/92631457/188595426-ad561e0c-4fdb-4d38-8804-26cb23d33c62.png)

    backend app
        balance     roundrobin
    {% for  host  in  groups['web'] %}
        server  app1  {{host}}:80  check
    {% endfor %}
    
 Now go to configuration file of haproxy --> /etc/haproxy/haproxy.cfg and copy the haproxy.cfg and paste inside /etc/ansible/roles/lbserver/templates. You can also give the location of configuration file of haproxy.

![image](https://user-images.githubusercontent.com/92631457/188595661-0df51dc2-3543-4ff2-9e6d-07643bda77d3.png)

    - name: "install HAPROXY"
      package:
         name: "haproxy"
         state: present
    - name: "copy HAPROXY configuration files to LoadBalancer"
      template:
         src: "haproxy.cfg"
         dest: /etc/haproxy/
      notify: lb restart
    - name: "restart HAPROXY"
      service:
         name: "haproxy"

         state: "started"
         
 ## STEP: 5
   - Now we are ready to run the ansible-playbook. go inside /etc/ansible/roles/projects.From here we can run the play book.

    ansible-playbook     playbook name

![image](https://user-images.githubusercontent.com/92631457/188596107-2c83795c-bdb8-4714-99ff-de9ada1ab58d.png)

![image](https://user-images.githubusercontent.com/92631457/188596166-c3555470-9bb3-4890-b0d2-dd66338ebe73.png)

## STEP: 6
  - Now we can check the weather our load balancer is working or not. Take public IP of load balancer with port 8080 (binding port)

![Screenshot 2022-09-06 at 2 45 07 PM](https://user-images.githubusercontent.com/92631457/188596878-8b89285a-ffc0-49e8-87b7-a1d858e5e6a0.png)

![Screenshot 2022-09-06 at 2 44 58 PM](https://user-images.githubusercontent.com/92631457/188596969-af39910a-c6a7-406a-84da-97bfdd1e7349.png)

![Screenshot 2022-09-06 at 2 44 32 PM](https://user-images.githubusercontent.com/92631457/188596991-2e0784a4-1e4f-47bf-8bad-9cc21e5a7bce.png)
