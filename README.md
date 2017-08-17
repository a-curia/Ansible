# Install
```
sudo apt-add-repository -y ppa:ansible/ansible
sudo apt-get update
sudo apt-get install -y ansible
```

# Managing Servers
```
sudo mv /etc/ansible/hosts /etc/ansible/hosts.orig
sudo vi /etc/ansible/hosts
```

- and add what you want
```
[web]
192.168.22.10
192.168.22.11
[local]
127.0.0.1
```

# Basic commands
```
ansible all -m ping
ansible all -m ping -s -k -u vagrant
```
  - all - Use all defined servers from the inventory file
  - -m ping - Use the "ping" module, which simply runs the ping command and returns the results
  - -s - Use "sudo" to run the commands
  - -k - Ask for a password rather than use key-based authentication
  - -u vagrant - Log into servers using user vagrant

# Modules
- uses "modules" to accomplish most of its Tasks. Modules can do things like install software, copy files, use templates and much more
- are the way to use Ansible, as they can use available context ("Facts") in order to determine what actions, if any need to be done to accomplish a Task
```
ansible all -s -m shell -a 'apt-get install nginx'
```

```
ansible all -s -m apt -a 'pkg=nginx state=installed update_cache=true'
```
  - all - Run on all defined hosts from the inventory file
  - -s - Run using sudo
  - -m apt - Use the apt module
  - -a 'pkg=nginx state=installed update_cache=true' - Provide the arguments for the apt module, including the package name, our desired end state and whether to update the package repository cache or not

# Playbook
- can run multiple Tasks and provide some more advanced functionality that we would miss out on using ad-hoc commands. Let's move the above Task into a playbook.
```
---
- hosts: local
  tasks:
   - name: Install Nginx
     apt: pkg=nginx state=installed update_cache=true
```

```
ansible-playbook -s nginx.yml
```

- use -s to tell Ansible to use sudo again, and then pass the Playbook file

# Handlers

- is exactly the same as a Task (it can do anything a Task can), but it will run when called by another Task. You can think of it as part of an Event system; A Handler will take an action when called by an event it listens for.
- is useful for "secondary" actions that might be required after running a Task, such as starting a new service after installation or reloading a service after a configuration change.

```
---
- hosts: local
  tasks:
   - name: Install Nginx
     apt: pkg=nginx state=installed update_cache=true
     notify:
      - Start Nginx

  handlers:
   - name: Start Nginx
     service: name=nginx state=started
```

# More Tasks
```
---
- hosts: local
  vars:
   - docroot: /var/www/myapp.com/public
  tasks:
   - name: Add Nginx Repository
     apt_repository: repo='ppa:nginx/stable' state=present
     register: ppastable

   - name: Install Nginx
     apt: pkg=nginx state=installed update_cache=true
     when: ppastable|success
     register: nginxinstalled
     notify:
      - Start Nginx

   - name: Create Web Root
     when: nginxinstalled|success
     file: dest={{ '{{' }} docroot {{ '}}' }} mode=775 state=directory owner=www-data group=www-data
     notify:
      - Reload Nginx

  handlers:
   - name: Start Nginx
     service: name=nginx state=started

    - name: Reload Nginx
      service: name=nginx state=reloaded
```

# Roles

- are good for organizing multiple, related Tasks and encapsulating data needed to accomplish those Tasks
- may involve adding a package repository, installing the package and setting up configuration
- once we start configuring our installations, the Playbooks tend to get a little more busy
- have a directory structure like this
```
rolename
 - files
 - handlers
 - meta
 - templates
 - tasks
 - vars
```
- within each directory, Ansible will search for and read any Yaml file called **main.yml** automatically.

### Files
- within the files directory, we can add files that we'll want copied into our servers, eg:
```
nginx
 - files
 - - h5bp
```

### Handlers
- we can put all of our Handlers that were once within the Playbook
- we can reference them from other files freely
```
---
- name: Start Nginx
  service: name=nginx state=started

- name: Reload Nginx
  service: name=nginx state=reloaded
```

### Meta
- contains Role meta data, including dependencies
- if this Role depended on another Role, we could define that here
```
---
dependencies:
  - { role: ssl }
```
- we can omit this file, or define the Role as having no dependencies
```
---
dependencies: []
```

### Template
- Template files can contain template variables, based on Python's Jinja2 template engine. Files in here should end in .j2, but can otherwise have any name. Similar to files, we won't find a main.yml file within the templates directory.
- example templates/myapp.com.conf
```
server {
    # Enforce the use of HTTPS
    listen 80 default_server;
    server_name *.{{ '{{' }} domain {{ '}}'  }};
    return 301 https://{{ '{{' }} domain {{ '}}'  }}$request_uri;
}

server {
    listen 443 default_server ssl;

    root /var/www/{{ '{{' }} domain {{ '}}'  }}/public;
    index index.html index.htm index.php;

    access_log /var/log/nginx/{{ '{{' }} domain {{ '}}'  }}.log;
    error_log  /var/log/nginx/{{ '{{' }} domain {{ '}}'  }}-error.log error;

    server_name {{ '{{' }} domain {{ '}}'  }};

    charset utf-8;

    include h5bp/basic.conf;

    ssl_certificate           {{ '{{' }} ssl_crt {{ '}}' }};
    ssl_certificate_key       {{ '{{' }} ssl_key {{ '}}' }};
    include h5bp/directive-only/ssl.conf;

    location /book {
        return 301 http://book.{{ '{{' }} domain {{ '}}'  }};
    }

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location = /favicon.ico { log_not_found off; access_log off; }
    location = /robots.txt  { log_not_found off; access_log off; }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;

        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_index index.php;

        include fastcgi_params; # fastcgi.conf for version 1.6.1+
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO       $fastcgi_path_info;
        fastcgi_param ENV production;
    }

    # Nginx status
    # Nginx Plus only
    #location /status {
    #    status;
    #    status_format json;
    #    allow 127.0.0.1;
    #    deny all;
    #}

    location ~ ^/(fpmstatus|fpmping)$ {
        access_log off;
        allow 127.0.0.1;
        deny all;
        include fastcgi_params; # fastcgi.conf for version 1.6.1+
        fastcgi_pass unix:/var/run/php5-fpm.sock;
    }
}
```

### Variables
- the vars directory contains a main.yml file which simply lists variables we'll use. This provides a convenient place for us to change configuration-wide settings.
```
---
domain: myapp.com
ssl_key: /etc/ssl/sfh/sfh.key
ssl_crt: /etc/ssl/sfh/sfh.crt
```
- three variables which we can use elsewhere in this Role. We saw them used in the template above, but we'll see them in our defined Tasks as well

### Tasks
```
---
- name: Add Nginx Repository
  apt_repository: repo='ppa:nginx/stable' state=present
  register: ppastable

- name: Install Nginx
  apt: pkg=nginx state=installed update_cache=true
  when: ppastable|success
  register: nginxinstalled
  notify:
    - Start Nginx

- name: Add H5BP Config
  when: nginxinstalled|success
  copy: src=h5bp dest=/etc/nginx owner=root group=root

- name: Disable Default Site
  when: nginxinstalled|success
  file: dest=/etc/nginx/sites-enabled/default state=absent

- name: Add SFH Site Config
  when: nginxinstalled|success
  register: sfhconfig
  template: src=myapp.com.j2 dest=/etc/nginx/sites-available/{{ '{{' }} domain {{ '}}'  }}.conf owner=root group=root

- name: Enable SFH Site Config
  when: sfhconfig|success
  file: src=/etc/nginx/sites-available/{{ '{{' }} domain {{ '}}'  }}.conf dest=/etc/nginx/sites-enabled/{{ '{{' }} domain {{ '}}'  }}.conf state=link


- name: Create Web root
  when: nginxinstalled|success
  file: dest=/var/www/{{ '{{' }} domain {{ '}}'  }}/public mode=775 state=directory owner=www-data group=www-data
  notify:
    - Reload Nginx

- name: Web Root Permissions
  when: nginxinstalled|success
  file: dest=/var/www/{{ '{{' }} domain {{ '}}'  }} mode=775 state=directory owner=www-data group=www-data recurse=yes
  notify:
    - Reload Nginx
```

- Add the nginx/stable repository
- Install & start Nginx, register successful installation to trigger remaining Tasks
- Add H5BP configuration
- Disable the default virtual host by removing the symlink to the default file from the sites-enabled directory
- Copy the myapp.com.conf.j2 virtual host template into the Nginx configuration
- Enable the virtual host configuration by symlinking it to the sites-enabled directory
- Create the web root
- Change permission for the project root directory, which is one level above the web root created previously

- There's some new modules (and new uses of some we've covered), including copy, template, & file. By setting the arguments for each module, we can do some interesting things such as ensuring files are "absent" (delete them if they exist) via state=absent, or create a file as a symlink via state=link.


### Running the Role
- before running the Role, we need to tell Ansible where our Roles are located
```
roles_path    = /vagrant/ansible/roles
```
- let's create a "master" yaml file which defines the Roles to use and what hosts to run them on -> server.yml
```
---
- hosts: all
  roles:
    - nginx
```
- and call the run
```
ansible-playbook -s server.yml
```

# Facts
- Before running any Tasks, Ansible will gather information about the system it's provisioning. These are called Facts, and include a wide array of system information such as the number of CPU cores, available ipv4 and ipv6 networks, mounted disks, Linux distribution and more.
Facts are often useful in Tasks or Tempalte configurations. For example Nginx is commonly set to use as any worker processors as there are CPU cores. Knowing this, you may choose to set your template of the nginx.conf file like so:
- single CPU
```
user www-data www-data;
worker_processes {{ ansible_processor_cores }};
pid /var/run/nginx.pid;
...
```

- multi CPU
```
user www-data www-data;
worker_processes {{ ansible_processor_cores * ansible_processor_count }};
pid /var/run/nginx.pid;
...
```

- all start with anisble_ and are globally available for use any place variables can be used: Variable files, Tasks, and Templates

