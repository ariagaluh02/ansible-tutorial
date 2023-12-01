# ANSIBLE CASE STUDY
#### `TESTED ON DOCKER`
#

## Docker Configuration
- Installed 3 **Debian OS** and 1 **CentOS 7**. 
- container **debian1** for workstation (Ansible Operator), **debian2** and **debian3** as a **Backend Group** and **centos1** as **Loadbalancer Group**.
- Using the same **docker network** to be able connect to each other container.
- IP Allocation:
  - debian1 = 172.22.0.3
  - debian2 = 172.22.0.2
  - debian3 = 172.22.0.4
  - centos1 = 172.22.0.5

## Workstation Configuration (debian1)
1. Update apt package 
    ```
    apt-get update -y
    ```

2. Install necessary package:
    ```
    apt-get install sudo git open openssh-server ansible nano iputils-ping -y
    ```    

3. Generate ssh key
    ```
    ssh-keygen -t rsa
    ```

4. Copy the content of id_rsa.pub to **~/.ssh/authorized_keys** location of each remote server 
    > NOTES: on each of the remote server must have first create the **authorized_keys** file inside ~/.ssh/

5. Create ansible directory at **/home/** and change directory to it
    ```
    mkdir /home/ansible
    cd /home/ansible
    ```

6. Create file **ansible.cfg** for ansible configuration file
    ```
    nano ansible.cfg
    ```
    ansible.cfg contain as follows:
    ```
    [defaults]
    inventory = inventory-tutorial
    ```

7. Create ansible **inventory** named **inventory-tutorial**
    ```
    nano inventory-tutorial
    ```
    configure as follows:
    ```
    [loadbalancer]
    node1 ansible_host=172.22.0.5

    [backend]
    node2 ansible_host=172.22.0.2
    node3 ansible_host=172.22.0.4
    ```
    > The node1 is centos1 container, node2 is debian2 container and node3 is debian3 container.

8. Create **playbook** directory to store all ansible-playbook file
    ```
    mkdir playbook
    ```

9. Create a playbook to **install nginx** on all server
    ```
    nano playbook/install_nginx.yaml
    ```
    configure as follows:
    ```
    ---
    - name: Install Nginx
      hosts: all
      become: true
      tasks:
        - name: Install Nginx on Debian
          apt:
            name: nginx
            state: present
          when: ansible_distribution == 'Debian'

        - name: Install repo epel-release
          yum:
            name: epel-release
            state: present
          when: ansible_distribution == 'CentOS'

        - name: Install Nginx on CentOS
          yum:
            name: nginx
            state: present
          when: ansible_distribution == 'CentOS'

        - name: Start Nginx Service
          service:
            name: nginx
            state: started
    ```
    The configuration above will install nginx on backend group server and loadbalancer group server. After nginx is installed, it will **start** the nginx service.

10. Create playbook to change the default nginx index page on all backend group server.
    ```
    nano playbook/change_index_nginx.yaml
    ```
    configure as follows:
    ```
    ---
    - name: Change Nginx Index Page on Backend Group
      hosts: backend
      become: true
      tasks:
        - name: Create custom index.html
          template:
            src: index.html.j2
            dest: /var/www/html/index.nginx-debian.html

        - name: Restart Nginx Service
          service:
            name: nginx
            state: restarted
    ```
    The configuration above will use jinja2 template that we will create next called **index.html.j2** and it will saved to each backend group destination. After it saved, the nginx service will restart on each backend group.

11. Create templates directory inside playbook directory, and create index.html.j2 template
    ```
    mkdir /playbook/templates
    nano  /playbook/templates/index.html.j2
    ```
    configure as follows:
    ```
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <p>served from: {{ inventory_hostname }}</p>
    </body>
    </html>
    ```

12. Create playbook to setup nginx in the loadbalancer group to be able to:
- load balance between nodes in the backend group
- Use "least connection" load balancing method
    ```
    nano playbook/setup_loadbalancer.yaml
    ```
    configure **setup_loadbalancer.yaml** as follows:
    ```
    ---
    - name: Setup Nginx Load Balancer
      hosts: loadbalancer
      become: true
      tasks:
        - name: Configure Nginx Load Balancer
          template:
            src: nginx_loadbalancer.conf.j2
            dest: /etc/nginx/conf.d/loadbalancer.conf
          notify:
            - restart nginx

      handlers:
        - name: restart nginx
          service:
            name: nginx
            state: restarted
    ```
    Create jinja template for loadbalancer group
    ```
    nano playbook/templates/nginx_loadbalancer.conf.j2
    ```
    Configure **nginx_loadbalancer.conf.j2** as follows:
    ```
    upstream backend {
      least_conn;
      server 172.22.0.2;
      server 172.22.0.4;
    }

    server {
      listen 80;
      server_name 172.22.0.5;

      location / {
        proxy_pass http://backend;
      }
    }
    ```

13. Check connection between server 
    ```
    ansible all -m ping
    ```
    > **Notes:** Need to install python3 and ssh already configured to be able to connected

14. Apply all of the ansible playbook
    ```
    ansible-playbook playbook/install_nginx.yaml
    ansible-playbook playbook/change_index_nginx.yaml
    ansible-playbook playbook/setup_loadbalancer.yaml
    ```

15. The next step is to check each server for each configuration change.
<br><br>

## Remote Server (Backed Group) Configuration (debian2 and debian3)
1. Update apt package & Install necessary package
    ```
    apt-get update && apt-get install sudo iputils-ping openssh-server python3 nano -y
    ```
2. Create file **authorized_keys** at ~/.ssh/
    ```
    nano ~/.ssh/authorized_keys
    ```
    change permission on **~/.ssh/** folder and **authorized_keys** file
    ```
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    ```
3. Copy the content of id_rsa.pub workstation (debian1) to **~/.ssh/authorized_keys**

4. Edit ssh configuration file
    ```
    nano /etc/ssh/sshd_config
    ```
    Open remark and configure as follows:
    ```
    PermitRootLogin yes
    PubkeyAuthentication yes
    ```
    restart ssh service
    ```
    service ssh restart
    ```    
    > For now, the worksation (debian1) should be able to ssh into the remote server and ansible able to ping to each remote server.

5. Check nginx installation
    ```
    service nginx status
    ```

6. Check the custom index nginx
    ```
    curl http://<ip-backend-remote-server>
    ```
    or we can check to our browser and type on the address bar:
    ```
    localhost:<container-port>
    ```

## Remote Server (Loadbalancer Group) Configuration (centos1)
1. Update yum package & Install necessary package
    ```
    yum update && yum install sudo iputils openssh-server python3 nano -y
    ```
2. Create file **authorized_keys** at ~/.ssh/
    ```
    nano ~/.ssh/authorized_keys
    ```
    change permission on **~/.ssh/** folder and **authorized_keys** file
    ```
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    ```
3. Copy the content of id_rsa.pub workstation (debian1) to **~/.ssh/authorized_keys**

4. Edit ssh configuration file
    ```
    nano /etc/ssh/sshd_config
    ```
    Open remark and configure as follows:
    ```
    PermitRootLogin yes
    PubkeyAuthentication yes
    ```
    restart ssh service
    ```
    systemctl restart sshd
    ```
    > For now, the worksation (debian1) should be able to ssh into the remote server and ansible able to ping to each remote server.

5. Check nginx installation
    ```
    service nginx status
    ```

6. Check the loadbalancer configuration nginx
    ```
    curl http://<ip-loadbalancer-group-server>
    ```
    it should showing:
    ```
    served from: node2 OR
    served from: node3
    ```
    It distributing incoming requests based on the least connection algorithm among the specified backend nodes. So, each time we make a request, it may go to either node2 or node3 based on their current connection loads.
