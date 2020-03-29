# Автоматизация администрирования. Ansible

## Задачи
Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере используя Ansible необходимо развернуть nginx со следующими условиями:
* необходимо использовать модуль yum/apt
* конфигурационные файлы должны быть взяты из шаблона jinja2 с перемененными
* после установки nginx должен быть в режиме enabled в systemd
* должен быть использован notify для старта nginx после установки
* сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible
* переделать оркестрацию на использование Ansible роли



## 1. Сборка playbook
В директории рядом с Vagrantfile создаю директории inventories, playbooks и templates
```
user@localhost:hw10-playbook$ mkdir {inventories,playbooks,templates}
```

Создаю файл inventory
```
user@localhost:hw10-playbook$ cat > ./inventories/nginx.yml
[web]
hw10-nginx ansible_host=127.0.0.1 ansible_port=2222 
ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```

Правлю ansible.cfg, указывая следующие настройки
```
inventory      = ./inventories/nginx.yml
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
```

Создаю playbook
```
user@localhost:hw10-playbook$ cat > ./playbooks/nginx.yml
---
- name: NGINX | Install and configure nginx
  hosts: hw10-nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install nginx package from EPEL Repo
      yum:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: ../templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration
		
	- name: NGINX | Create test page
      command: echo Hello ansible > /usr/share/nginx/html/index.html
      tags:
        - nginx-testpage


  handlers:
    - name: start nginx
      systemd:
        name: nginx
        state: started
        enabled: yes

    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes

    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded

```

Создаю шаблон для конфига nginx
```
user@localhost:hw10-playbook$ cat > ./templates/nginx.conf.j2
events {
  worker_connections 1024;
}
http {
  server {
    listen {{ nginx_listen_port }} default_server;
    server_name default_server;
    root /usr/share/nginx/html;
    location / {
    }
  }
}

```

Добавляю в Vagrantfile секцию провижионинга ansible следом за секцией shell-провижионинга:
```
box.vm.provision :ansible do |ansible|
		    ansible.playbook = "./playbooks/nginx.yml"
		  end
```

Запускаю образ и проверяю с хостовой машины:
```
user@localhost:hw10-playbook$ vagrant up
user@localhost:hw10-playbook$ curl 192.168.11.150:8080
Hello ansible
```

## 2. Миграция playbook на role

Копирую Vagrantfile и ansible.cfg в новую директорию
```
mkdir ../hw10-role
cp Vagrantfile ansible.cfg ../hw10-role
```

Формирую структуру директорий для roles
```
user@localhost:hw10-playbook$ cd ../hw10-role
user@localhost:hw10-role$ mkdir -p ansible/roles/hw10-nginx/{handlers,tasks,templates,vars}
```

Создаю файл inventory и playbook
```
user@localhost:hw10-role$ cat > ./ansible/inventory.yml
[web]
hw10-nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key

user@localhost:hw10-role$ cat > ./ansible/playbook.yml
---
- name: Role config
  hosts: web
  gather_facts: false
  roles:
    - hw10-nginx
  become: true

```

Создаю таски
```
user@localhost:hw10-role$ cat > ./ansible/roles/hw10-nginx/tasks/main.yml
---
- name: NGINX | Install EPEL Repo package from standart repo
  yum:
    name: epel-release
    state: present
  tags:
    - epel-package
    - packages

- name: NGINX | Install nginx package from EPEL Repo
  yum:
    name: nginx
    state: latest
  notify:
    - restart nginx
  tags:
    - nginx-package
    - packages

- name: NGINX | Create NGINX config file from template
  template:
    src: ../templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify:
    - reload nginx
  tags:
    - nginx-configuration
	
		
- name: NGINX | Create test page
  command: echo Hello ansible > /usr/share/nginx/html/index.html
  tags:
  - nginx-testpage


```

Создаю хэндлеры
```
user@localhost:hw10-role$ cat > ./ansible/roles/hw10-nginx/handlers/main.yml
---
- name: start nginx
  systemd:
    name: nginx
    state: started
    enabled: yes

- name: restart nginx
  systemd:
    name: nginx
    state: restarted
    enabled: yes

- name: reload nginx
  systemd:
    name: nginx
    state: reloaded
	
```

Шаблон конфига для nginx
```
user@localhost:hw10-role$ cat > ./ansible/roles/hw10-nginx/templates/nginx.conf.j2
events {
  worker_connections 1024;
}
http {
  server {
    listen {{ nginx_listen_port }} default_server;
    server_name default_server;
    root /usr/share/nginx/html;
    location / {
    }
  }
}

```

Заношу переменную
```
user@localhost:hw10-role$ cat > ./ansible/roles/hw10-nginx/vars/main.yml
---
    nginx_listen_port: 8080

```

Вношу правки в ansible.cfg
```
inventory      = ./inventories/ __=>__ inventory      = ansible/inventory.yml
#roles_path __=>__ roles_path    = ./ansible/roles
```

В Vagrantfile секцию ansible-провижионинга заменяю следующим
```
box.vm.provision :ansible do |ansible|
		    ansible.playbook = "./ansible/playbook.yml"
			ansible.inventory_path = "./ansible/inventory.yml"
		  end
```

Проверяю:
```
user@localhost:hw10-role$ vagrant up
user@localhost:hw10-playbook$ curl 192.168.11.150:8080
Hello ansible
```