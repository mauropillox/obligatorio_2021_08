Crear un LAMP stack e implementar aplicacion usando Ansible Playbooks.
-------------------------------------------

Cosas a tener en cuenta dado que estamos trabajando en un CentOS 8:

1. iptables es reemplazado por firewalld
2. MySQL s reemplazado por MariaDB

Debajo se define el inventory:

[webservers]
#192.168.1.11
192.168.1.16 (Ubuntu)

[dbservers]
192.168.1.11 (CentOS {Contolador Ansible})

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
    when: ansible_facts['os_family'] == "RedHat"
  
  - name: Install ntp
    apt: name=ntp state=present
    tags: ntp/ntp
    when: ansible_facts['os_family'] == "Ubuntu"

  - name: Install common dependencies
    apt:
      name: 
        - firewalld
      state: present
    when: ansible_facts['os_family'] == "Ubuntu"

Instala chrony o ntp dependiendo del OS Family, en caso de CentOS, instala Chrony, en caso de Ubuntu, instala NTP.

A diferencia del playbook original, se modifican los paquetes de instalacion a utilizar asi como el state, que previamente estaba en installed, lo cual era incorrecto.


El siguiente handler ubicado en: "obligatorio_2021_08/lamp/roles/common/handlers/main.yml"

 - name: restart chronyd
    service: name=chronyd state=restarted

Reinicia el servicio de chronyd

Web:

En: "obligatorio_2021_08/lamp/roles/web/tasks/install_httpd.yml" se encuentra el siguiente playbook: 

---
 - name: Install httpd and php
    apt: 
      name: httpd 
      state: present
    with_items:
     - httpd
     - php
     - php-mysql
     - firewalld
    when: 
     ansible_facts['os_family'] == "Ubuntu"

  - name: Install Git
    apt: 
      name: git
      state: present
    with_items:
    - git
    when: 
     ansible_facts['os_family'] == "Ubuntu"

Ejecuta un apt para instalar https y php en el Webserver de Ubuntu, asi tambien como Git.

A diferencia del playbook original, se modifican los paquetes de instalacion a utilizar (se usa apt) asi como el state. Tambien se agrega al with_items el servicio firewalld

A continuacion se inician los servicios de firewalld previamente instalados y se adaptan a Ubuntu, se agrega el become:yes para evitar problemas de permisos root, el state se cambia a enabled, a diferencia del playbook original.

  - name: Start firewalld
    service: 
      name: firewalld 
      state: enabled
      enabled: yes
      become: yes
    when: 
     ansible_facts['os_family'] == "Ubuntu"
    
  - name: insert firewalld rule for httpd
    firewalld: 
      port: 443/tcp 
      permanent: true 
      state: enabled
      immediate: yes
    become: yes
    when: 
     ansible_facts['os_family'] == "Ubuntu"

  - name: http service state
    service: 
      name: httpd
      state: enabled
      enabled: yes
    when: 
     ansible_facts['os_family'] == "Ubuntu"
      
  - name: Configure SELinux to allow httpd to connect to remote database
    seboolean: 
      name: httpd_can_network_connect_db 
      state: yes 
      persistent: yes
    when: 
     ansible_facts['os_family'] == "Ubuntu"


DB:

El siguiente playbook ubicado en "obligatorio_2021_08/lamp/roles/common/db/main.yml, se encuentra el siguiente playbook para la instalaccion de MariaDB sus usuarios y permisos

  - name: Install MariaDB package
    yum: 
      name: 
        - mariadb-server 
        - python3-PyMySQL
      state: present

A diferencia del playbook original, se cambia el state a "present" y se configuran los paquetes adecuados para el CentOS, ya que el mismo correra el database server.

  - name: Configure SELinux to start mysql on any port
    seboolean: name=mysql_connect_any state=true persistent=yes

Se configura el modulo de seguridad de SELinux para arrancar en cualquier puerto para iniciar mySQL

  - name: Create Mysql configuration file
    template: src=/home/ansible/ansible/obligatorio_2021_08/lamp/roles/db/templates/my.cnf.j2 dest=/var/lib/mysql
    notify:
    - restart mariadb

Pasa la configuraciones de my.cnf.j2 del archivo de configuracion de MySQL al destino indicado. A diferencia del playbook original, se uso el path correcto que el source era el template guardado en db/templates y el destino se puso como /var/lib/mysql asi la informacion esta en ese archivo.

  - name: Create MariaDB log file
    file: path=/var/log/mysqld.log state=touch owner=mysql group=mysql mode=0775

  - name: Create MariaDB PID directory
    file: path=/var/run/mysqld state=touch owner=mysql group=mysql mode=0775

  - name: Start MariaDB Service
    service: name=mariadb state=started enabled=yes

  - name: Start firewalld
    service: name=firewalld state=started enabled=yes

Crea carpeta de log y asigna permisos de usuario owner.

  - name: insert firewalld rule
    firewalld: port={{ mysql_port }}/tcp permanent=true state=enabled immediate=yes

Inserta regla de firewall con el puerto de mySQL a utilizar el cual esta cargado en una variable global en dbservers llamada mysql_port. 

  - name: Create Application Database
    mysql_db: 
      name={{ dbname }} 
      state=present 
      config_file=/home/ansible/ansible/obligatorio_2021_08/lamp/roles/db/templates/my.cnf.j2

Create aplicacion de base de datos, se le agrega un archivo de configuracion para que tome de referencia el cual es el template guardado en db/template que es un my.cnf.j2.

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

Crea el usuario MySQL usando las variables encontradas en dbservers:

mysqlservice: mysqld
mysql_port: 3306
dbuser: foouser
dbname: foodb
upassword: abc

Para no usar config_file, se le pasan las variables globales de la base de datos en el mismo role

El Handler, ubicado en: obligatorio_2021_08/lamp/roles/db/handlers/main.yml

- name: restart mariadb
  service: name=mariadb state=restarted

Reinicia el servicio MariaDB en caso que sea llamado
