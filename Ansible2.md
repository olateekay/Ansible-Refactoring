# Ansible Refactoring & Static Assignments (Imports and Roles)


In this project you will continue working with `ansible-config-mgt` repository and make some improvements of your code. Now you need to refactor your Ansible code, create assignments, and learn how to use the imports functionality. Imports allow to effectively re-use previously created playbooks in a new playbook - it allows you to organize your tasks and reuse them when needed.


## Step 1 - Jenkins job enhancement

Before we begin, let us make some changes to our Jenkins job - now every new change in the codes creates a separate directory which is not very convenient when we want to run some commands from one place. Besides, it consumes space on Jenkins serves with each subsequent change. Let us enhance it by introducing a new Jenkins project/job - we will require Copy Artifact plugin.

1. Go to your Jenkins-Ansible server and create a new directory called `ansible-config-artifact` - we will store there all artifacts after each build.

`sudo mkdir /home/ubuntu/ansible-config-artifact`

2. Change permissions to this directory, so Jenkins could save files there -

 `chmod -R 0777 /home/ubuntu/ansible-config-artifact`

3. Go to Jenkins web console -> Manage Jenkins -> Manage Plugins -> on Available tab search for Copy Artifact and install this plugin without restarting Jenkins

4. Create a new Freestyle project  and name it `save_artifacts`.
This project will be triggered by completion of your existing ansible project. Configure it accordingly:


*Note*: You can configure number of builds to keep in order to save space on the server, for example, you might want to keep only last 2 or 5 build results. You can also make this change to your ansible job.

5. The main idea of save_artifacts project is to save artifacts into `/home/ubuntu/ansible-config-artifact` directory. To achieve this, create a Build step and choose Copy artifacts from other project, specify ansible as a source project and `/home/ubuntu/ansible-config-artifact` as a target directory.

6. Test your set up by making some change in `README.MD` file inside your `ansible-config-mgt` repository (right inside master branch).
If both Jenkins jobs have completed one after another - you shall see your files inside `/home/ubuntu/ansible-config-artifact` directory and it will be updated with every commit to your master branch.


![alt text](image1.jpg)

![alt text](image2.jpg)

```

ubuntu@ip-172-31-44-66:~$ cd /home/ubuntu/ansible-config-artifact
ubuntu@ip-172-31-44-66:~/ansible-config-artifact$ ls
LICENSE  README.md  ansible.md  images  inventory  playbooks

```


## Step 2 - Refactor Ansible code by importing other playbooks into `site.yml`


Let see code re-use in action by importing other playbooks.

1. Within `playbooks` folder, create a new file and name it `site.yml` - This file will now be considered as an entry point into the entire infrastructure configuration. Other playbooks will be included here as a reference. In other words, `site.yml` will become a parent to all other playbooks that will be developed. Including `common.yml` that you created previously. 

2. Create a new folder in root of the repository and name it `static-assignments`. The `static-assignments` folder is where all other children playbooks will be stored. This is merely for easy organization of your work. 

3. Move `common.yml` file into the newly created `static-assignments` folder.

4. Inside `site.yml` file, import `common.yml` playbook.

```
- hosts: all
- import_playbook: ../static-assignments/common.yml

```

The code above uses built in `import_playbook `Ansible module.

The folder structure should look like this;

```
├── static-assignments
│   └── common.yml
├── inventory
    └── dev
    └── stage
    └── uat
    └── prod
└── playbooks
    └── site.yml
```

5. Run ansible-playbook command against the dev environment
Since you need to apply some tasks to your dev servers and wireshark is already installed - you can go ahead and create another playbook under static-assignments and name it `common-del.yml`. In this playbook, configure deletion of wireshark utility.

```
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    yum:
      name: wireshark
      state: removed

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    apt:
      name: wireshark-qt
      state: absent
      autoremove: yes
      purge: yes
      autoclean: yes

```

update `site.yml` with -
 `import_playbook: ../static-assignments/common-del.yml ` instead of `common.yml` and run it against dev servers:

```
sudo ansible-playbook -i /home/ubuntu/ansible-config-mgt/inventory/dev /home/ubuntu/ansible-config-mgt/playbooks/site.yml

```

Output:

```
ubuntu@ip-172-31-44-66:~/ansible-config-mgt$  ansible-playbook -i /home/ubuntu/ansible-config-mgt/inventory/dev /home/ubuntu/ansible-config-mgt/playbooks/site.yml

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [172.31.15.118]
ok: [172.31.11.45]
ok: [172.31.13.254]
ok: [172.31.40.150]
ok: [172.31.5.122]

PLAY [update web, nfs and db servers] ******************************************

TASK [Gathering Facts] *********************************************************
ok: [172.31.11.45]
ok: [172.31.15.118]
ok: [172.31.13.254]
ok: [172.31.5.122]

TASK [delete wireshark] ********************************************************
changed: [172.31.5.122]
changed: [172.31.11.45]
changed: [172.31.13.254]
changed: [172.31.15.118]

PLAY [update LB server] ********************************************************

TASK [Gathering Facts] *********************************************************
ok: [172.31.40.150]

TASK [delete wireshark] ********************************************************
changed: [172.31.40.150]

PLAY RECAP *********************************************************************
172.31.11.45               : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
172.31.13.254              : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
172.31.15.118              : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
172.31.40.150              : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
172.31.5.122               : ok=3    changed=1    unreachable=0    failed=0   skipped=0    rescued=0    ignored=0


```


## Step 3 - Configure UAT Webservers with a role ‘Webserver’

We have our nice and clean `dev` environment, so let us put it aside and configure 2 new Web Servers as `uat`. We could write tasks to configure Web Servers in the same playbook, but it would be too messy, instead, we will use a dedicated role to make our configuration reusable.

1.Launch 2 fresh EC2 instances using RHEL 8 image, we will use them as our uat servers, so give them names accordingly  `Web1-UAT` and `Web2-UAT`.

2.To create a role, you must create a directory called `roles/`, relative to the playbook file or in `/etc/ansible/ directory`

There are two ways how you can create this folder structure:

Use an Ansible utility called `ansible-galaxy` inside `ansible-config-mgt/roles` directory (you need to create roles directory upfront)

```
mkdir roles
cd roles
ansible-galaxy init webserver

```

Note: You can choose either way, but since you store all your codes in GitHub, it is recommended to create folders and files there rather than locally on `Jenkins-Ansible` server.

The entire folder structure should look like below, but if you create it manually - you can skip creating `tests`, `files`, and `vars` or remove them if you used `ansible-galaxy`

```

└── webserver
    ├── README.md
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml

```


After removing unnecessary directories and files, the roles structure should look like this:

```
└── webserver
    ├── README.md
    ├── defaults
    │   └── main.yml
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    └── templates

```

Update your inventory `ansible-config-mgt/inventory/uat.yml` file with IP addresses of your 2 UAT Web servers:

```
[uat_webservers]
uat-1 ansible_host=<Web1-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user'
uat-2 ansible_host=<Web1-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user'

```

Run a ping, to see if everything is wired correctly

`$ ansible all -i /home/ubuntu/ansible-artifact/inventory/uat -m ping`


```
Output

uat-2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
uat-1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}

```

In `/etc/ansible/ansible.cfg` file uncomment roles_path string and provide a full path to your roles directory roles_path = `/home/ubuntu/ansible-config-mgt/roles`, so Ansible could know where to find configured roles.

It is time to start adding some logic to the webserver role. Go into tasks directory, and within the main.yml file, start writing configuration tasks to do the following:

- Install and configure Apache (httpd service)
- Clone Tooling website from GitHub https://github.com//tooling.git.
- Ensure the tooling website code is deployed to /var/www/html on each of 2 UAT Web  servers.
- Make sure httpd service is started

```
---
- name: install apache
  become: true
  ansible.builtin.yum:
    name: "httpd"
    state: present

- name: install git
  become: true
  ansible.builtin.yum:
    name: "git"
    state: present

- name: clone a repo
  become: true
  ansible.builtin.git:
    repo: https://github.com/<your-name>/tooling.git
    dest: /var/www/html
    force: yes

- name: copy html content to one level up
  become: true
  command: cp -r /var/www/html/html/ /var/www/

- name: Start service httpd, if not started
  become: true
  ansible.builtin.service:
    name: httpd
    state: started

- name: recursively remove /var/www/html/html/ directory
  become: true
  ansible.builtin.file:
    path: /var/www/html/html
    state: absent

```

## Step 4 - Reference ‘Webserver’ role

Within the `static-assignments` folder, create a new assignment for `uat-webservers` `uat-webservers.yml`. This is where you will reference the role.

```
---
- hosts: uat-webservers
  roles:
     - webserver

```

Remember that the entry point to our ansible configuration is the `site.yml` file. Therefore, we need to refer your `uat-webservers.yml` role inside `site.yml`.

So, we should have this in `site.yml`

```
---
- hosts: all
- import_playbook: ../static-assignments/common.yml

- hosts: uat-webservers
- import_playbook: ../static-assignments/uat-webservers.yml

```

## Step 5 - Commit & Test

Commit your changes, create a Pull Request and merge them to master branch. Ensure the webhook triggers two consequent Jenkins jobs, they ran successfully and copied all the files to your `Jenkins-Ansible `server into `/home/ubuntu/ansible-artifact/ directory.`
Now run the playbook against your `uat` inventory and see what happens:

`ansible-playbook -i /home/ubuntu/ansible-artifact/inventory/uat /home/ubuntu/ansible-artifact/playbooks/site.yml`


```
PLAY [uat_webservers] *********************************************************************************************************************************************************
TASK [Gathering Facts] ********************************************************************************************************************************************************
ok: [uat-2]
ok: [uat-1]
TASK [webserver : install apache] *********************************************************************************************************************************************
changed: [uat-1]
changed: [uat-2]
TASK [webserver : install git] ************************************************************************************************************************************************
changed: [uat-2]
changed: [uat-1]
TASK [webserver : clone a repo] ***********************************************************************************************************************************************
changed: [uat-2]
changed: [uat-1]
TASK [webserver : copy html content to one level up] **************************************************************************************************************************
changed: [uat-2]
changed: [uat-1]
TASK [webserver : Start service httpd, if not started] ************************************************************************************************************************
changed: [uat-2]
changed: [uat-1]
TASK [webserver : recursively remove /var/www/html/html/ directory] ***********************************************************************************************************
changed: [uat-2]
changed: [uat-1]
PLAY RECAP ********************************************************************************************************************************************************************
uat-1                      : ok=8    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
uat-2                      : ok=8    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

You should be able to see both of your UAT Web servers configured and you can try to reach them from your browser:

![alt text](image3.jpg)

Your Ansible architecture now looks like this:



![alt text](image4.jpg)