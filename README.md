Crear un LAMP stack e implementar aplicacion usando Ansible Playbooks.
-------------------------------------------

Cosas a tener en cuenta dado que estamos trabajando en un CentOS 8:

1. iptables es reemplazado por firewalld
2. MySQL s reemplazado por MariaDB

Debajo se define el inventory:

[webservers]
192.168.1.11 ansible_user=ansible ansible_password=ansible01

[dbservers]
192.168.1.11 ansible_user=ansible ansible_password=ansible01

En este caso ambos los playbooks correran para todo los webservers y database, los respectivos playbooks para estos roles, asi como un common dependancies en los demas hosts. Para ejecutar el playbook:

        ansible-playbook -i hosts site.yml

Para verificar se podria ir a http://192.168.1.11/index.php

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

En la ruta: "obligatorio_2021_08/lamp/roles/" se encuentran los distintos roles y su respectivo uso. Los 3 roles son:

Common: 

El siguiente playbook ubicado en "obligatorio_2021_08/lamp/roles/common/tasks/main.yml"

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

Reinicia el servicio de chronyd

Web:

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

Se instala Git.

  - name: Start firewalld
    service: 
      name: firewalld 
      state: started
      enabled: yes

Inicia el servicio de firewalld

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

Verifica que el servicio httpd este prendido

  - name: Configure SELinux to allow httpd to connect to remote database
    seboolean: 
      name: httpd_can_network_connect_db 
      state: yes 
      persistent: yes

Configura el modulo de seguridad para el kernel de linux (SELinux)

DB:

El siguiente playbook ubicado en "obligatorio_2021_08/lamp/roles/common/db/main.yml, se encuentra el siguiente playbook para la instalaccion de MariaDB sus usuarios y permisos

  - name: Install MariaDB package
    yum: 
      name: 
        - mariadb-server 
        - python3-PyMySQL
      state: present

Instala MariaDB

  - name: Configure SELinux to start mysql on any port
    seboolean: name=mysql_connect_any state=true persistent=yes

  - name: Create Mysql configuration file
    template: src=/home/ansible/ansible/obligatorio_2021_08/lamp/roles/db/templates/my.cnf.j2 dest=/var/lib/mysql
    notify:
    - restart mariadb

Pasa la configuraciones de my.cnf.j2 al destino indicado 

  - name: Create MariaDB log file
    file: path=/var/log/mysqld.log state=touch owner=mysql group=mysql mode=0775

  - name: Create MariaDB PID directory
    file: path=/var/run/mysqld state=touch owner=mysql group=mysql mode=0775

  - name: Start MariaDB Service
    service: name=mariadb state=started enabled=yes

  - name: Start firewalld
    service: name=firewalld state=started enabled=yes

Crea carpeta de log y asigna permisos de usuario owner

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

Crea la aplicacion de base de datos y el usuario cuyas configuraciones estan en las variables de: "obligatorio_2021_08/lamp/group_vars/dbservers"

mysqlservice: mysqld
mysql_port: 3306
dbuser: foouser
dbname: foodb
upassword: abc

El Handler, ubicado en: obligatorio_2021_08/lamp/roles/db/handlers/main.yml

- name: restart mariadb
  service: name=mariadb state=restarted

Reinicia el servicio MariaDB en caso que sea llamado
