En: "obligatorio_2021_08/lamp/site.yml" se encuentra el playbook que invoca a los distintos roles:

	- hosts: all
	  remote_user: ansible
	  roles:
	   - common

	- hosts: webservers
	  remote_user: ansible
	  roles:
	    - web

	- hosts: dbservers
	  remote_user: ansible
	  roles:
	    - db

En la ruta: "obligatorio_2021_08/lamp/roles/" se encuentran los distintos roles y su respectivo uso. 

common: 

La siguiente task, ubicada en "obligatorio_2021_08/lamp/roles/common/tasks/main.yml"

  - name: Install Chrony
    yum: name=chrony state=present
    tags: ntp/chrony

  - name: Install common dependencies
    yum:
      name: 
            - firewalld
      state: present

  - name: Configure ntp file
    template: 
      src: /home/ansible/ansible/obligatorio_2021_08/lamp/roles/common/templates/ntp.conf.j2
      dest: /etc/ntp.conf
    tags: ntp
    notify: restart chronyd

Instala y configura chrony tomando las configuraciones de ntp.conf.j2 y la dependencia firwalld:

driftfile /var/lib/ntp/drift

restrict 127.0.0.1 
restrict -6 ::1

server 192.168.1.11

includefile /etc/ntp/crypto/pw

keys /etc/ntp/keys

El siguiente handler ubicado en: "obligatorio_2021_08/lamp/roles/common/handlers/main.yml"

 - name: restart chronyd
    service: name=chronyd state=restarted

reinicia el servicio de chronyd


DB:

La siguiente task, ubicada en "obligatorio_2021_08/lamp/roles/common/db/main.yml, se encuentra el siguiente playbook para la instalaccion de mardiaDB sus usuarios y permisos

  - name: Install MariaDB package
    yum: 
      name: 
        - mariadb-server 
        - python3-PyMySQL
      state: present

instala mariadb

  - name: Configure SELinux to start mysql on any port
    seboolean: name=mysql_connect_any state=true persistent=yes

  - name: Create Mysql configuration file
    template: src=/home/ansible/ansible/obligatorio_2021_08/lamp/roles/db/templates/my.cnf.j2 dest=/var/lib/mysql
    notify:
    - restart mariadb

pasa la configuraciones de my.cnf.j2 al destino indicado 

  - name: Create MariaDB log file
    file: path=/var/log/mysqld.log state=touch owner=mysql group=mysql mode=0775

  - name: Create MariaDB PID directory
    file: path=/var/run/mysqld state=touch owner=mysql group=mysql mode=0775

  - name: Start MariaDB Service
    service: name=mariadb state=started enabled=yes

  - name: Start firewalld
    service: name=firewalld state=started enabled=yes

crea carpeta de log y asigna permisos de usuario owner

  - name: insert firewalld rule
    firewalld: port={{ mysql_port }}/tcp permanent=true state=enabled immediate=yes

  - name: Create Application Database
    mysql_db: 
      name={{ dbname }} 
      state=present 
      config_file=/home/ansible/ansible/obligatorio_2021_08/lamp/roles/db/templates/my.cnf.j2

  - name: Create Application DB User
    mysql_user: 
      check_implicit_admin=true
      name={{ dbname }} 
      login_user={{ dbuser }}
      login_password={{ upassword }} 
      password={{ upassword }}
      priv=*.*:ALL 
      host='%' 
      state=present
      #config_file=/home/ansible/ansible/obligatorio_2021_08/lamp/roles/db/templates/my.cnf.j2

Crea la aplicacion de base de datos y el usuario cuyas configuraciones estan en: obligatorio_2021_08/lamp/roles/db/templates/my.cnf.j2

[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

symbolic-links=0
port={{ mysql_port }}

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

Tambien se toman algunas variables ubicadas en: "obligatorio_2021_08/lamp/group_vars/dbservers"

mysqlservice: mysqld
mysql_port: 3306
dbuser: foouser
dbname: foodb
upassword: abc


El Handler, ubicado en: obligatorio_2021_08/lamp/roles/db/handlers/main.yml

- name: restart mariadb
  service: name=mariadb state=restarted

reinicia el servicio mariadb

web:


En: "obligatorio_2021_08/lamp/roles/web/tasks/install_httpd.yml" se encuentra el siguiente playbook: 

---
  - name: Install httpd and php
    yum: 
      name: php 
      state: present
    with_items:
     - httpd
     - php
     - php-mysql

Ejecuta un yum para instalar https y php

  - name: Install web role specific dependencies
    yum: 
      name: git
      state: present
    with_items:
    - git

  - name: Start firewalld
    service: 
      name: firewalld 
      state: started
      enabled: yes

se inicia firewalld

  - name: insert firewalld rule for httpd
    firewalld: 
      port: 443/tcp 
      permanent: true 
      state: enabled
      immediate: yes

Se habilita el puerto 443 para tcp

  - name: http service state
    service: 
      name: httpd
      state: started
      enabled: yes

  - name: Configure SELinux to allow httpd to connect to remote database
    seboolean: 
      name: httpd_can_network_connect_db 
      state: yes 
      persistent: yes

Configura el modulo de seguridad para el kernel de linux (SELinux)
