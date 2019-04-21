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

#TODO TALK ABOUT KNOWING HOW SERVERS WORK

#TODO TALK ABOUT SSH (OR WINDOWS SOMETHING)?

# Need to talk about the Control Machine VS server set up. Where do you run Ansible, on what? Requiring what? Good place to talk about idempotence

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

# TODO Do I need something here about Idempotence? Do I do that with handlers bit?

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

        * src - what file do you want copied from your control machine to the remote machine

        * dest - where on the remote machine should the file be put

        * owner: what user on the remote machine should be the owner

        * mode: what permissions does the file need on the remote machine

    * [https://docs.ansible.com/ansible/latest/modules/copy_module.html#copy-module](https://docs.ansible.com/ansible/latest/modules/copy_module.html#copy-module)

* File

    * Sets attributes of files

    * https://docs.ansible.com/ansible/latest/modules/file_module.html#file-module

* package | yum | apt

* git

* mysql_user

#### More advanced usage of tasks 

1. loop 

2. Module outputs (register)

3. changed?

#### Docs for modules 

By Category - [https://docs.ansible.com/ansible/latest/modules/modules_by_category.html](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)

One big list [https://docs.ansible.com/ansible/latest/modules/list_of_all_modules.html](https://docs.ansible.com/ansible/latest/modules/list_of_all_modules.html)

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

