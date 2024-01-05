# How to deploy Flask application using Ansible in local VM

1. Install a Ubuntu Machine in VM
    1. Using Vagrantfile create a Ubuntu machine and install Python Ansible and Docker
        ```vagrantfile
        Vagrant.configure("2") do |config|
          # Master node
          config.vm.define "ansible-master" do |master|
            master.vm.box = "ubuntu/bionic64"
            master.vm.network "forwarded_port", guest: 22, host: 2222
            master.vm.network "forwarded_port", guest: 80, host: 8080
            master.vm.network "forwarded_port", guest: 443, host: 8443
            master.vm.network "private_network", type: "static", ip: "192.168.50.10"
            master.ssh.insert_key = false
            master.vm.provider "virtualbox" do |v|
              v.memory = 1024
            end
            master.vm.provision "shell", inline: <<-SHELL
              sudo apt-get update
              sudo apt-get install -y python ansible
              sudo echo ansible-master > /etc/hostname
              sudo hostnamectl set-hostname ansible-master
            SHELL
          end

          # Worker nodes
          (1..2).each do |i|
            config.vm.define "worker#{i}" do |worker|
              worker.vm.box = "ubuntu/bionic64"
              worker.vm.network "private_network", type: "static", ip: "192.168.50.#{i + 10}"
              worker.ssh.insert_key = false
              worker.vm.provider "virtualbox" do |v|
                v.memory = 512
              end
              worker.vm.provision "shell", inline: <<-SHELL
                sudo apt-get update
                sudo apt-get install -y python
                sudo echo worker#{i} > /etc/hostname
                sudo hostnamectl set-hostname worker#{i}
              SHELL
            end
          end
        end

        ```
        Once running you can ssh it using `vagrant ssh ansible-machine`
        
3. Generate ssh key from your master node that need to copy in all worker node

    Now copy the `id_rsa.pub` using ssh-keygen and copy to all server by `ssh-copy-id root@172.17.0.2`, this will help login Ansible during deployment.
    
    I re-confirm by by `ssh root@172.17.0.2` without giving password.

    Create a new file in your home directory name `inventory.txt`, e.g.
        
        db_and_web_server1 
        db_and_web_server2 

    Now create the playbook.yml file
    
    ```yml
    - name: Deploy a web application
      hosts: db_and_web_server1,db_and_web_server2
      become: yes

      tasks:
        - name: Upgrade pip
          pip:
            name: pip
            extra_args: --upgrade

        - name: Install all required dependencies
          apt: name={{ item }} state=present
          with_items:
            - python 
            - python-setuptools 
            - python-dev 
            - build-essential 
            - python-pip 
            - python-mysqldb

        - include: tasks/deploy_db.yml
        - include: tasks/deploy_web.yml
    ```

    Now create the deploy_db.yml file  ***(make sure you keep the same alignment)*
    ```yml
        - name: Install and Configure Database
          apt: name={{ item }} state=present
          with_items:
            - mysql-server 
            - mysql-client

        - name: Start MySQL service
          service: 
            name: mysql
            state: started
            enabled: yes

        - name: Create Application Database
          mysql_db: name='{{ db_name }}' state=present

        - name: Create Database User
          mysql_user:
            name: '{{ db_user }}'
            password: '{{ db_pass }}'
            priv: '*.*:All'
            state: present

        # - name: Execute MySQL query
          # mysql_db:
            # login_unix_socket: /var/run/mysqld/mysqld.sock
            # login_user: db_user
            # login_password: Passw0rd
            # login_db: employee_db
            # query: "INSERT INTO employees VALUES ('JOHN');"
    ```
    
    Now create the deploy_db.yml file ***(make sure you keep the same alignment)*
    ```yml
        - name: Install Python Flask dependency
          pip: name={{ item }} state=present
          with_items:
            - flask 
            - flask-mysql 

        - name: Copy Source code
          copy: src=app.py dest=/opt/app.py

        - name: Start web server
          shell: "cd /opt && FLASK_APP=app.py nohup flask run --host=0.0.0.0 --port=5000 > /var/log/flask.log 2>&1 &"
    ```
    Also create the `host_vars` files as given in the source. We can use the group vars but here i use the host specific vars

7. Now run the ansible command as below
    `ansible-playbook playbook.yml -i inventory.txt`
    
    ![](https://i.imgur.com/ryfhlxJ.png)

8. Once done check from your browser with your worker URL
        http://192.168.50.11:5000                            => Welcome
        http://192.168.50.11:5000/how%20are%20you            => I am good, how about you?
        http://192.168.50.11:5000/read%20from%20database     => JOHN

9. There is one issue with ansible, you can not insert query, create table etc.. directly!!, but you can import query as dump. So here manualy update the DB as below on both the Worker. *(I know its not the best way to implement, here i just demonstrate the Ansible feature)*
    
    ```sql
    mysql> CREATE DATABASE employee_db;
    mysql> GRANT ALL ON *.* to db_user@'%' IDENTIFIED BY 'Passw0rd';
    mysql> USE employee_db;
    mysql> CREATE TABLE employees (name VARCHAR(20));
    mysql> INSERT INTO employees VALUES ('JOHN');
    
    ```
