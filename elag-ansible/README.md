## Welcome

Hello, I’m Chad Nelson from Temple University Libraries in Philadelphia, USA. I actually started my Library Technology career in the UK, at London Metropolitan University, but I only learned about ELAG just before I moved back to the states, so I’m excited to be here for my first ELAG.

Anyway, today I’m going to be teaching you about Ansible, a tool for automating server builds, application deployments, and orchestrating changes in your infrastructure.

The plan for the session is to start with an introduction to Ansible concepts and resources, and then walk through building a fully functional Ansible playbook to build a working copy of Catmandu configured with some predefined options. After that, we’ll refactor that playbook to create a Catmandu role for Ansible that will be easily reusable across projects, and make it publishable as an open, reusable component that anyone can use. We’ll also touch on some best practices like testing.

But before we do that, I’d like to go around the room and have everyone quickly introduce themselves and mention what they hope to get out of this session.

(5 minutes of intros)

## Intro to Ansible

High Level Ansible Introductions

So, what is Ansible. To quote Wikipedia, 

"An ansible is a category of fictional device or technology capable of near-instantaneous communication." Wait, sorry, not that’s the description of the term from science fiction, as the term ansible originated with author Ursula Le Guin’s “Hainish Cycle” which included her famous books the Dispossessed and The Left Hand of Darkness. It is a device used to communicate across the cosmos. 

Our need for ansible is a little bit more terrestrial, so, let me instead quote the the Ansible docs:

"Ansible is an IT automation tool...Ansible’s main goals are simplicity and ease-of-use. It also has a strong focus on security and reliability, featuring a minimum of moving parts, usage of OpenSSH for transport...and a language that is designed around auditability by humans–even those not familiar with the program.”

But that’s a high level abstraction, and I think we want to start much lower.

At its core, Ansible is a Python library and executable (or really, libraries and executables) that run a sequence of tasks based on the configuration your provide it via (mostly) yaml files. Ansible is also a set of practices that are encouraged, though not necessarily enforced, by the expectations of the executable.

Now, while Ansible is written in Python, you don’t actually need to know much about Python to use it. In the 5 plus years I’ve been using Ansible, I have maybe had to really dig into the Python parts of it only a handful of times, usually to troubleshoot issues that ended up being bugs that I reported or fixed in the core library. 

Basically, all you need to know is how to install Python on a machine, you’ve reached your limit of required Python knowledge to use Ansible. Now, knowing more about Python is certainly going to be useful for really excelling with Ansible. When you get into things like writing automated tests or improving developer experience by standardizing python build requirements, som knowledge of working with Python will be useful.


#TODO Need to talk about the Control Machine VS server set up. Where do you run Ansible, on what? Requiring what? Good place to talk about idempotence

## Ansible Concepts

### Intro

So, now we’re going to talk about lower level Ansible concepts, the parts you will put together to make Ansible do the things you want it to do. We’re going to talk about 

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

* A name - 

    * Not technically required, but strongly suggested. This is big part of the set of practices that make up Ansible to try to make your playbooks and roles comprehensible to no specialists. This is where you describe what and maybe why something is happening.

* A module

    * This is where we leverage Ansible’s huge built in library of modules to cover most common tasks. A module consists of the module name, and then module parameters that determine what that module should do. I think this makes the most sense through a series of examples

#### Examples of common modules

* copy 
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

* package
    * Manage packages from your OS's package manager
    * Basic parameters
        * name - name of the package you want to install
        * state - should it be present, absent, latest?
            * Idempotence is import here - these translate to:
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
 ```
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
```
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

```
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

```
- name: Add librarian and circultation users to book and patron db
  mysql_user:
    name: "{{ item[0] }}"
    priv: "{{ item[1] }}.*:ALL"
    password: "foo"
  loop: "{{ ['librarian', 'circulation'] |product(['books', 'patrons'])|list }}"
```  

  The array syntax looks a little different, but ultimately, we're combining our arrays to allow our tasks to iterate over every combination of user and database.

  Ansible provides a large number of ways to filter and combine your data for more complicated data tasks, but that is outsode the scope of this introduction.

##### when

##### become

##### register

##### changed_when



### Variables

#### What are variables for

How they are used (transform an earlier example of yum package installation from directly listing them inline to putting them in a variable)

#### Where are variables defined

#### Vault for Encryption

### Templates

### Handlers

### Inventory

#### Hosts

##### Groups

##### Hosts

#### Dynamic Inventory

### Plays

### Playbooks

### Roles

#### What is a role

#### Ansible Galaxy

## Break

## Build a playbook

## Build a better playbook with a role
