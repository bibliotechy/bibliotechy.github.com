## Welcome

Hello, I’m Chad Nelson from Temple University Libraries in Philadelphia, USA. I actually started my Library Technology career in the UK, at London Metropolitan University, but I only learned about ELAG just before I moved back to the states, so I’m excited to be here for my first ELAG.

Anyway, today I’m going to be teaching you about Ansible, a tool for automating server builds, application deployments, and orchestrating changes in your infrastructure.

The plan for the session is to start with an introduction to Ansible concepts and resources, and then walk through building a fully functional Ansible playbook to build a working copy of Catmandu configured with some predefined options. After that, we’ll refactor that playbook to create a Catmandu role for Ansible that will be easily reusable across projects, and make it publishable as an open, reusable component that anyone can use. We’ll also touch on some best practices like testing.

But before we do that, I’d like to go around the room and have everyone quickly introduce themselves and mention what they hope to get out of this session.

(5 minutes of intros)

## Intro to Ansible

High Level Ansible Introductions

So, what is Ansible. To quote Wikipedia, 

"An ansible is a category of fictional device or technology capable of near-instantaneous communication." Wait, sorry, that’s the description of the term from science fiction, as the term ansible originated with author Ursula Le Guin’s “Hainish Cycle” which included her famous books the Dispossessed and The Left Hand of Darkness. It is a device used to communicate across the cosmos. 

Our need for ansible is a little bit more terrestrial, so, let me instead quote the the Ansible docs:

"Ansible is an IT automation tool...Ansible’s main goals are simplicity and ease-of-use. It also has a strong focus on security and reliability, featuring a minimum of moving parts, usage of OpenSSH for transport...and a language that is designed around auditability by humans–even those not familiar with the program.”

But that’s a high level abstraction, and I think we want to start much lower.

At its core, Ansible is a Python library and executable (or really, libraries and executables) that run a sequence of tasks based on the configuration you provide it via (mostly) yaml files. Ansible is also a set of practices that are encouraged, though not necessarily enforced, by the expectations of the executables.

Now, while Ansible is written in Python, you don’t actually need to know much about Python to use it. In the 5 plus years I’ve been using Ansible, I have maybe had to really dig into the Python parts of it only a handful of times, usually to troubleshoot issues that ended up being bugs that I reported or fixed in the core library.

Basically, all you need to know is how to install Python on a machine, you’ve reached your limit of required Python knowledge to use Ansible. Now, knowing more about Python is certainly going to be useful for really excelling with Ansible. When you get into things like writing complicated templates, automated tests, or improving developer experience by standardizing python build requirements, some knowledge of working with Python will be useful.

## Ansible Concepts

### High Level

#### Control and Target

So, at a high level, Ansible works on a "control machine / target machine(s)" model. Control Machine is where your Ansible code and configuration lives. On the control machine you run the ansible binaries and it then communicates with the target machine(s) over ssh (and scp and sftp), sending the instructions to the target machines which then runs them locally. So, both your control machine and the target machine need to have Python, whic is not usually a problem for linux and similar machines.

One of Ansible's main goals is that playbooks should be able to run on a machine multiple times, and always have the same result. As in , if Ansible is configured to make sure that a line is in a file, you should be able to run the ansible code multiple times, but that line only shows up in the file once. Ansible calls this "idempotency" and it is a core part of what Ansible modules deliver.

### Intro

So, now we’re going to talk about lower level Ansible concepts, the parts you will put together to make Ansible do the things you want it to do. We’re going to talk about 

* 

* Tasks

* Variables

* Templates

* Handlers

* Inventory

* Plays & Playbooks

* Roles

### Task

#### What is a task?

So, let’s start with the most basic unit of work in Ansible, the Task. A task is how you dowload a file, change ownership, install a package, or almost any of the other basic tasks you need to set the state of your system. 

A minimal task is composed of

* **A name**: Not technically required, but strongly suggested. This is big part of the set of practices that make up Ansible to try to make your playbooks and roles comprehensible to no specialists. This is where you describe what and maybe why something is happening.

* **A module**: This is where we leverage Ansible’s huge built in library of modules to cover most common tasks. A module consists of the module name, and then module parameters that determine what that module should do. I think this makes the most sense through a series of examples

#### Examples of common modules

##### copy

  * Copy files to a remote location 
  * Basic parameters
      * src - file to copy from control machine to remote machine
      * dest - where on the remote machine should the file be put
      * owner: what user on the remote machine should be the owner
      * mode: what permissions does the file need on the remote machine
  * ```
      - name: Copy config.xml into place
        copy:
          src: config.xml
          dest: /opt/myapp/config.xml
          owner: myuser
          mode: 0644
      ```

  * [More Parameters and examples](https://docs.ansible.com/ansible/latest/modules/copy_module.html#copy-module)

##### package

* Manage packages from your OS's package manager
* Basic parameters
  * name - name of the package you want to install
  * state - should it be present, absent, latest? Idempotence is import here - these translate to:
    * present - install once, never touch again
    * latest - check every time for updates
    * If you  goal is maintaining consistency across deployment environments, this can get tricky. You might deploy different package versions to production because an update came out since you last deployed. 
    * My personal recommendation is to pin specific versions of critical packages, which is done via the name `name: httpd-2.4.8`
  
* ```
  - name: Install nginx
    package:
      name: nginx
      state: latest
  ```
  * ```
    - name: Install nginx
      package:
        name: nginx-1.15.9
        state: present
    ```
  * [More Parameters and examples](https://docs.ansible.com/ansible/latest/modules/package_module.html#package-module)
  * If you need to do more OS specific work, like building apt cache, you can use the [apt](https://docs.ansible.com/ansible/latest/modules/apt_module.html#apt-module) or [yum](https://docs.ansible.com/ansible/latest/modules/yum_module.html#yum-module) modules to get more granular. 
* service
    * Manage services no matter the underlying *nix OS service manager
        * Supported init systems include BSD init, OpenRC, SysV, Solaris SMF, systemd, upstart.
    * Basic paramters
        * name - name of the service to manage
        * state - one of
            * reloaded
            * restarted
            * started
            * stopped
        * enabled - should this service be enabled at startup?
    * ```
      - name: Make sure httpd is started and set to run on server startup
        service:
          name: httpd
          state: started
          enabled: true
      ```
    * [More parameter and examples](https://docs.ansible.com/ansible/latest/modules/service_module.html#service-module)

* mysql_db
    * Manage mysql databases on a remote machine
    * Basic paramters
        * name: name of the database you want to manage
        * state: one of:
            * present
            * absent
            * dump - dump out that database to a file (requires additional `target` parameter)
            * import - Create db and import data based on a file    

    * ```

      - name: Create marc records database
        mysql_db:
          name: marcrecords
          state: present
      ```
    * ```    
      - name: Copy database dump file
        copy:
          src: dump.sql.bz2
          dest: /tmp
      - name: Import the db dump
        mysql_db:
          name: my_db
          state: import
          target: /tmp/dump.sql.bz2
      ```
    * [More parameters and examples](https://docs.ansible.com/ansible/latest/modules/mysql_db_module.html#mysql-db-module)

#### More Modules

##### More essential modules
* [command](https://docs.ansible.com/ansible/latest/modules/command_module.html#command-module) - run arbitrary shell commands
* [git](https://docs.ansible.com/ansible/latest/modules/git_module.html#git-module) - manage git repositories on remote machines
* [stat](https://docs.ansible.com/ansible/latest/modules/stat_module.html#stat-module) - check for presence of files / directories

##### Full listing of modules
[By Category](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)

[One big list](https://docs.ansible.com/ansible/latest/modules/list_of_all_modules.html)


####  Advanced Task Usage 

##### loop
loops are ways to repeat multiple actions using a single task. Think of a for loop in programming languages. You provide the loop keyword an array, and the Ansible handles substituting it in place of the `{{ item }}` variable.
The simplest loop looks like:
 ```yaml
- name: Install nginx
  package:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - libxslt
    - ImageMagick
```

In addition to arrays of strings, we can also pass in arrays of dictionaries and access the keys. Let's say we want to create two users, `librarian` and `circulation` for a mysql database called `books`. The `mysql_user` module can create database users and give them permissions to access databases. A task might look like:
```yaml
- name: Add librarian and circultation users to books db
  mysql_user:
    name: "{{ item.name }}"
    priv: books.*:ALL
    password: "{{ item.password }}"
  loop:
    - {name: "librarian", password: "shhhhhhhhh"}
    - {name: "circulation", password: "youhavefines"}
```


This is our first look at variables, which we will get into in more detail soon. But at this point it is important to know that variables are interepreted as Jinja templates by Ansible. Jinja is a tempalting library widely used in Python, and is notable here for it's filters which can be used to transform arrays to do more complex operations.


Now, going back to our example mysql users, let's say our two users, `librarian` and `circulation` need access to two databases, `books` and `patrons` . Using a basic loop like we did above, we could just have two tasks, one for each database

```yaml
- name: Add librarian and circultation users to books db
  mysql_user:
    name: "{{ item.name }}"
    priv: books.*:ALL
    password: "{{ item.password }}"
  loop:
    - {name: "librarian", password: "shhhhhhhhh"}
    - {name: "circulation", password: "youhavefines"}

- name: Add librarian and circultation users to patron db
  mysql_user:
    name: "{{ item.name }}"
    priv: patrons.*:ALL
    password: "{{ item.password }}"
  loop:
    - {name: "librarian", password: "shhhhhhhhh"}
    - {name: "circulation", password: "youhavefines"}
```

But that's duplicating code and data, and becomes a hassle to maintain. Luckily, Ansible can handle more complex arrays using Jinja filters.

```yaml
- name: Add librarian and circultation users to book and patron db
  mysql_user:
    name: "{{ item[0] }}"
    priv: "{{ item[1] }}.*:ALL"
    password: "foo"
  loop: "{{ ['librarian', 'circulation'] |product(['books', 'patrons'])|list }}"
```  

  The array syntax looks a little different, but ultimately, we're combining our arrays to allow our tasks to iterate over every combination of user and database.

  Ansible provides a large number of ways to filter and combine your data for more complicated data tasks, but that is outsode the scope of this introduction. [Ansible's documentaion on loops](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#standard-loops) is thorough and useful if you want to dig deeper.

##### register and when

`register` and `when` are two optional attributes of a task that provide flow control and conditional logic in ansible. Then can be used separately, but you they are often used in conjunction.

`register` stores the output of a task into a variable for later use.

`when` determines if a task should be run based on some input

For example, if we have a app with a custom install script that needs to be run the first time, but not after that, you'll need to check that it has not already been run. Built in ansible commands do this autoamtically, that's the "idempotence" that is built in. Often you'll need to think through a proxy that tells you a script has been run. Maybe a directory is created during the installation process, and so you can assume that if the directory exists, the script has run.

```yaml
- name: Check if the custom install script has already run
  stat:
    path: /opt/myapp/data
  register: myapp_installed

- name: Run the custom installation if required
  command:>
    /tmp/myapp/bin/install
  when: myapp_installed.stat.isdir
```

So here, the first task checks for the presence of the direcotry. The second task runs the installations but only when the directory doesn't exist.

##### become and become_user

Ansible alos provides the ability to run tasks as privileged (sudo) users and as arbitrary users with the `become` and `become_user` commands. `become` is often applied to a group of tasks in a play, which we will talk about to later, but I'll show an example here of using it in a task.

IN this example, we want to make sure that our `index.html` file is owned by our app user. Assigning file ownership often requires a privileged account, and so we use the `become: true` attribute.
```yaml
- name: Make sure the file is owned by the app user
  file:
    path: /opt/myapp/public/index.html
    owner: app_user
  become: true
```

`become_user` is useful when a command needs to be run a particular user, for example:

```yaml
- name: Run the sync rake task
  command:>
    bundle exec rake sync
  become: true
  become_user: app_user
```

### Variables

#### What are variables

Variables are the ways

"Variable names should be letters, numbers, and underscores. Variables should always start with a letter."

Variables can be:
* strings - `ssh_port: 33`
* lists -
```yaml
firewall_ports:
  - 33
  - 80
  - 443
```
* dictionaries - 
```yaml
ruby_versions:
  default_version: 2.5.1
  versions
```
* vault value - a special kind of string - more later


#### Where are variables defined?

This is a complicated topic. Because variables are the backbone of flexibility in Ansible, they can be defined in numerous places. Two we've already seen are:

register variables
```yaml
- name: Is the install already run?
  stat:
    path: /opt/my_app/install.txt
  register: my_app_installed
```

loop variables:
```yaml
- name: install some packages
  package:
    name: "{{ item }}"
    state: present
  loop:
   - nginx
   - fail2ban
```

But variables can also be defined in:

* role default directories
* group_vars directories
* inventory host files
* inventory group_vars
* standalone files that are imported
* playbooks
* pass them in at the command line

Ansible also has a [detailed precedence order](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable) that is useful for considering how to make your ansible work flexible and reusable.

One use of this precedence might be to allow overriding defaults. For example, going back to our loop example

```yaml
- name: install some packages
  package:
    name: "{{ item }}"
    state: present
  loop:
   - nginx
   - fail2ban
```

Those two packages are hard coded, making this task only able to do one thing. If we instead defined the packages as a variable, it could then be later overriden. We'll talk a little bit more about that when we get to inventories.

```yaml
# vars/main.yml
webapp_dependencies:
  - nginx
  - fail2ban

- name: install some packages
  package:
    name: "{{ item }}"
    state: present
  loop: "{{ webapp_dependencies }}"
```

#### Facts

So, as we have seen, you can define variables that indicate packages and versions and passwords, but ansible also provides a very powerful set of dynamic variables that ansible discovers about you target machines, and these are called facts.

Facts include information such as

* Operating System and Version
* ipv4 and ipv6 addresses

You can use these facts to dynamically configure target machines based on information from other machines. For example, you might need to add a configuration to your webapp detailing the ip address of the database the app will use. Rather than hard coding that into a variable, using facts to dynamically set that value provides flexibility when new machines are provisioned and works seamlessly across inventories (QA, Stage, Production) without requiring extra manual variable creation.

If our inventory looks like

```text
# inventory/hosts
[app]
app1 ansible_ssh_address=123.456.789.101 user=root

[db]
db1 ansible_ssh_address=987.654.321.123 user=root
```

Then our task can use that defined

```yaml
- name: set the db connection string
  lineinfile:
    dest: /home/my_user/.bashrc
    regexp: '^export MYAPP_DB_CONNECTION='
    line: "export MYAPP_SECRET_SECRET=http://{{ host[db1].ip_address }}:3306/my_app_db"
```

MORE ABOUT FACTS

#### Vault for Encryption
Now, obviously, some of the variables you might want to use to configure your Ansible tasks will be sensitive - passwords, api keys, etc. Ansible provides a built in tool for handling these kinds of information - `ansible-vault`.

With vault, you can encrypt a single variable, or an entire entire file, and then store it directly in your playbook. Ansible will then handle seamlessly decrypting those variables and files in any task. You need to provide the same encryption key at both encryption time and playbook runtime.

Let's look at an example:

Say we have a yaml file of passwords we want to copy to a specific place on the target server.

```
---

secret_secret: "i've got a secret"
```

We can encrypt that whole file with:

```bash
$ ansible-vault encrypt password.yml --ask-vault-pass
> New Vault password:
> Confirm New Vault password:
> Encryption successful
```

Which turns that file into:

```text
$ANSIBLE_VAULT;1.1;AES256
35393639393366386637646339303537646437353330393433386430666365356364316161383637
3733306539663236626433343562616232376335333539390a643334616538353738346638336464
62336433386234303235393666633564336663303430313031313131386336663231646439623266
3366663634373837360a373631366238376665323361363437326562393566663838373434383936
3735
```

And then we use the file as normal:

```yaml
- name: copy passwords file into place
  copy:
    src: passwords.yml
    dest: /opt/my_app/config/passwords.yml
```

And when we look at it on the target server, it looks like the original:

```bash
$ cat /opt/my_app/config/passwords.yml
>
---

secret_secret: "i've got a secret"
```

While useful, the biggest issue I find here is that if the password encrypted in the file cannot be re-used elsewhere.  `ansible-vault` provides a more granular way to solve this issue, which is `encrypt_string`

```bash
$ ansible-vault encrypt_string -n my_app_secret_secret "ive got a secret"
> 
my_app_secret_secret: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          34643162353362633239313237303137643730653934343934626238636164383435393834396661
          3765336134336638306238303139616434653530396531620a366361643761646265343334313264
          37653833353638323438613961626134376261636265323436653432666135616130386532303862
          6630626431366161350a393733383438316333333533306463623233373534303831316531306133
          3634
```

You can then copy that text output directly into a variables file, and it can be used like any other variable:

```yaml
- name: make sure the password is defined as an environment variable
  lineinfile:
    dest: /home/my_user/.bashrc
    regexp: '^export MYAPP_SECRET_SECRET='
    line: "export MYAPP_SECRET_SECRET={{ my_app_secret_secret }}"

- name: add mysql user
  mysql_user: 
    name: my_user
    password: "{{ my_app_secret_secret }}
    databse: myapp
    perms: *:ALL
```

That means that the the password can be reused in other places, like the data base user creation and configuration for the client that will talk to the database with that user. From a maintainability standpoint, it also makes it easier to grep through your files wen you need to update something. When the variable name is encrypted, it becomes harder.

### Templates

Templates are essentially special case variables that use all of the features of Jinja, the python template language. You will use it to create a file that needs to differ each time it is created on your system. 

They are invoked by a task called `template` which needs to know what is the template source, and where does the template need to go on the remote machine. Sources are expected to have the be in one of a few `templates` directories, and have the`.j2` suffix, which inidates it is a jinja file.

```yaml
my_app_user: app_user_1
my_app_db_driver: mysqli
my_app_force_ssl: false
```

```yaml
 - name: Add app config
   template:
     src: config.ini.j2
     dest: /opt/my_app/config.ini
```

```ini
# templates/config.ini.j2
[application]
user={{ my_app_user }}
db_driver={{ my_app_db_diver }}
{% if my_app_force_ssl %}
force_ssl=true
{% endif %}
```

The output would be

```ini
[application]
user=app_user_1
db_driver=mysqli
```

### Handlers

Handlers are special case tasks that are only supposed to run at the end of a group of tasks, otherwise known as a play, which we will come back to later.

Essentially, any task can "notify" a handler that it needs to be run, and Ansible will collect those notifications and run them at the end of that group of tasks.

I think this is best understood with an example:

```yaml
- name: Add my_app apache vhost config
  template:
    src: app-vhosts.conf.j2
    dest: /etc/httpd/conf.d/app.conf
  notify: restart httpd

- name: Make non ssl redirect to ssl
  file:
    src: ssl-redirect.conf
    dest: /etc/httpd/conf.d/ssl-redirect.yml
  notify: restart httpd
```

With both of these tasks, they need apache to be restarted in order for the changes to take place. Handlers allow you to efficiently register a task that needs to happen, but wait till all the tasks that might need it to happen have run, and then do the thing.

The other really nice this about handlers is that they only run when needed. Tasks respond with whether or not they "changed" something on the target machine each time they run. Say, the first time you run these above tasks, they are going to create the conf files on the remote machine. So hey have changed somethign. But the second time you run this playbook, it won't have to make any chnaages cause the required files will already been in place. So, that second run, the task will respond to Ansible with `changed: false`.

Handlers are onyl notified whe the task returns `changed: true`. So in our case, Ansible won't restart httpd every time you run it, it will ony restart it when an actualy chnage has occured that requires httpd to be restarted.

```yaml
# handlers/main.yml

- name: restart httpd
  service:
    name: httpd
    status: restarted
```

### Inventory

Inventory is where you define target machines, but it also provides an opportunity for a lot of flexibility.

The main file is the `hosts` file.

An inventory `hosts` can define:
* machine
* machine variables 
* groups
* group variables

```text
# inventory/vagrant/hosts

[app]
app.mydomain.com ansible_ssh_port=2222

[database]
db.mydomain.com ansible_ssh_port=2222``
```

Additionally, you can also define multiple inventories, if you want to have a QA, Staging, and Production version of your system, having an inventory for each would allow you to acheive that.

```text
[app]
app.mydomain.com

[mysql]
mysql.mydomain.com

[mongo]
mongo.mydomain.com

[db]
mysql
mongo

```

Having multiple inventory allows you to set up parallele structures for your environments, abstracting their differences away so that you run the same tasks on each environment.
```
inventory/
├── prod
│   └── hosts
├── qa
│   └── hosts
└── vagrant
    └── hosts
```


### Plays

Plays are groups of tasks that are performed on a set of target machines. Usually, plays use groups defined in the `hosts` file to determine what work needs to be done.

Just like tasks, plays get a name. You can use `become` in a play so that all tasks performed in this play will be run with that escalated privileges.

At it's simplest, you can just include tasks directly in a play

```
- name: DB Server installation
  hosts: app
  become: true
  tasks:
    - name: install mysql
      package:
        name: mysql-server
        state: present
    - name: start the mysql service
      service:
        name: mysql
        state: started
        enable: true
```

But more often, plays are used in conjunction with roles, which we will talk about later, rather than directly inlining the tasks. Roles, as we will see, encapsulate a group of functionality together to make it reuasable.

```yaml
- name: DB Server installation
  hosts: app
  become: true
  roles:
    - mysql
```
#### Pre and Post tasks
Sometimes, though, a role is a little too general, and you some extra tasks to make it work the right way, and Ansible provides `pre_tasks` and `post_tasks` for this purpose. `pre_tasks` happen before the roles are run, and `post_tasks` run afterwards.

```yaml
- name: App server installation
  hosts: app
  become: true
  pre_tasks:
    - name: create my_app user to run app
      user: my_app_user_special
      state: present
  roles:
    - httpd
    - my_app
  post_tasks:
    - name: add extra config line
      lineinfile:
        line: "enable_threads=true"
        file: /opt/my_app
```

They allow you to continue to use generic roles, so long as the extra required work only needs to take place before or after those roles.

##### Parameterized roles

One other very powerful way to work with your role here is to parameterize your role that is included in the play.

```yaml
- name: App server installation
  hosts: app
  become: true
  roles:
    - role: httpd
      force_ssl: true
      max_workers: 128
    - my_app
```

I'm going to editorialize here a little bit. One really nice thing about including the vars that are relevant to your play directly in the play via paramterized roles, is that it becomes very easy to understand what the play is ultimately doing. As we talked about earlier, variables can be defined in many places, so there can often be a disconect between the generic tasks being run and the specific variables defined somewhere else. For a certian level of complexity, this is the right approach, as it eases readability. 

On the other hand, inlining like this makes it very hard to overrideand use the flexibility. If your production env has 4 cores per app server, but your QA only has one, you probably do need different number of workers for apache, or different heap space allocation.


#### Addtional play params

Plays can also take additional parameters including:

* when: conditional logic for all the included tasks
* vars: yet another place to define a variable
* handler: define an inline hanlder if not provided elsewhere
* remote_user: what user needs to run these. Defaults to root

### Roles

So, as we touched on earlier, roles are an abstraction layer for Ansible, allowing you to group together related tasks, handlers, and variables, into a logic, reusable unit.

#### What is in a role

A role is a directory with sub directoires in a particular structure that Ansible expects.

The most important part of a role are:
* tasks
* defaults
* files
* templates
* handlers

```text.
├── README.md
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

To me, the most important part of a role is parameterizing your tasks effectively and setting reasonable defaults.

Parameterizing tasks provides opportunities for users of the role to override what they need. In this way, the paramters are the interface into your role, allowing users to ge the output they want. 

But it is important to make that easy and intuiitive. For example if you have an application install directory, and then some options as to where thigns are installed within that, you will want to next variable. For example, you coudl start with:

```yaml
---

app_install_directory: /opt/my_app
app_data_path: /opt/my_app/data

```
So, while this allows users to override at a granular level, they woudl need to override both variables. If they just override installation directory, they are going to be pretty surprised where their data directory end up.

So the next iteration might be:

```yaml
---

app_install_directory: /opt/my_app
app_data_path: "{{ app_install_directory}}/data"

```

Which improves the situation. But, it still leaves them needing have to copy over your original defaults and then update it. To override `app_data_path`, they would need to define their own variable as
`app_data_path: "{{ app_install_directory}}/data"`

What I tend to prefer is adding another variable like:

```yaml
---

app_install_directory: /opt/my_app
app_date_dirname: "data"
app_data_path: "{{ app_install_directory }}/{{ app_data_dirname }}"

```

So that overriding becomes much cleaner.

```
app_install_directory: /opt/other_location
app_data_dirname: my_data
```

#### Meta

One additonal piece of information you deinfe in a roles is the metadata, which is where you define any other roles your role depends on (which will be automatically run before you the role that defines it as a dependency), the OS's it supports, license, author, etc.

#### Ansible Galaxy

Ansible Galaxy is the primary way we take advantage of those reusable roles, particularly the ones written by other people. AG is actually a command line excecutable and a website.

The website, which I will leave to you to explore, allows anyone to publish their roles for others to use, and helps you find them based on the OS you need to support, and names.


The command line tool is very useful, as it allows you to 

#### Install a role from galaxy 

```bash
ansible-galaxy install tulibraries.shibboleth_sp
```

#### Install a bunch of roles defined in a `requirements.yml` file

`requirement.yml`

```yml
- src: geerlingguy.apache
  version: 3.0.3

- src: zzet.rbenv
  version: 3.4.2

- src: tulibraries.ansible_role_passenger_apache
  version: v0.2
```

```bash
ansible-galaxy install -r requirements.yml
```

#### Create a new role with the structure defined

`ansible-galaxy init my_app`

### Playbooks

So, playbooks take all of the stuff we have learned and expose them in a way that we can use. A playbook is actually just a series of one of more plays in a single file.

```yaml
- name: Bootstrap everything
  hosts: all
  become: true
  roles:
    - bootstrap

- name: Install db
  hosts: db
  roles:
    - mysql

- name: install my_app
  hosts: app
  roles:
    - my_app
```

By convention, your main playbook is named `playbook.yml`, but you can have multiple playbooks that each do one thing, so you can call them separately, but then compose them into a main playbook with `import_playbook` command.


```yaml
# bootstrap.yml

- name: Bootstrap everything
  hosts: all
  become: true
  roles:
    - bootstrap
```

```yaml
# db.yml

- name: Install db
  hosts: db
  roles:
    - mysql
```

```yaml
# app.yml

- name: install my_app
  hosts: app
  roles:
    - my_app
```

```yaml
# playbook.yml

import_playbook: bootstrap.yml
import_playbook: db.yml
import_playbook: app.yml
```

## Break

## Build a playbook


## Build a better playbook with a role
