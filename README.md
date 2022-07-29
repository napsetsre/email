# Sending Mail with Ansible

## Sending email on the internet
This lab is dependent on you have an SMTP server available.  If you have never worked with mail servers before, configuring them to work versus configuring them to be trusted on the internet is an enormous difference.  It's usually not hard to install and begin using an email server.  It's far more challenging for that email server to be trusted on the internet. It's not uncommon for the well-known ports used to send email to be blocked altogether by ISP's and cloud providers in order to present roadblocks to spammers.

A much better option is to leverage a SMTP service such as Sendgrid, Mailgun, or AWS Simple Email Service (SES).  This labe was tested with AWS SES specifically, but most services that offer a means to send email allow a fairly high number of messages to be sent for free (often 10,000/day or more).  Alternatively, free e-mail services such as GMail will often allow you to use their SMTP servers with an account you've signed up for.

Setting up your own trusted SMTP server is a non-trivial task.  If you're not addressing a privacy or scalability concern, it's highly recommended to leverage a service instead if you need to send mail across the internet.

## Variables
host: URL for target email service
username: Username for email service
password: password for email service
email_source: Email address the mail should come from
email_dest: List of email addresses to send to


## Lab 1
The first lab is a simple playbook with the variables baked in.  In order to prevent secret exposure by hardcoding the password, we leverage an Ansible lookup to search for an environment variable instead.  This is great for local testing, but will struggle to scale as everyone running the playbook would need to know this password in order to run the playbook.
```
---
- hosts: localhost
  connection: local
  
  vars:
    host: email-smtp.us-east-2.amazonaws.com
    username: AKIAQY43FAYYIBN6X52W
    password: "{{ lookup('env', 'EMAIL_PASSWORD') }}"
    email_source: test@dragonslair.dev
    email_dest:
    - personone@dragonslayer.dev
    - persontwo@dragonslayer.dev
    - dragonslairpark@gmail.com

  tasks:
  - name: Sending an e-mail using AWS Simple Email Service servers
    community.general.mail:
      host: "{{ host }}"
      port: 587
      username: "{{ username }}"
      password: "{{ password }}"
      from: test@dragonslair.dev
      to: "{{ email_dest }}"
      subject: Ansible-report
      body: "System {{ ansible_hostname }} has been successfully provisioned."

```

### Running the playbook
If you have ansible installed locally, you could run the playbook from the 1-playbook directory with
```
ansible-playbook playbook.yml
```

This will work if you have the Ansible installed with the correct Python packages and Ansible collections.  On most RHEL-based systems, this isn't overly complicated.  `yum install ansible` with `ansible-galaxy collection install community.general` is generally all that's needed.  However, you may find that on a system that has been around for a while and been used for a lot of different tasks that there are conflicting library versions or the version of community.general is far older.  One way to solve this is through use of Python virtual environments.  However, when working as part of an automation team it's still a non-trivial task to ensure everyone is sharing the same environment to run playbooks from.

One solution as Ansible and containers have matured is to run Ansible within a container.  These are referred to as execution environments.  Two tools are available to make Ansible specific environments simpler to use.

#### Ansible Builder - https://www.ansible.com/blog/introduction-to-ansible-builder
Ansible Builder can be installed via pip:
```
pip install ansible-builder
```

You can also build an execution environment like you would any other container through definition files such as Dockerfiles.  The advantage to using Ansible Builder is a simpler way to define the specific collections, Python libraries, and packages that are necessary for your playbook without having to understand the intricacies of a Containerfile (Dockerfile).  If you prefer the lower-level Containerfile, you could also use Ansible-Builder to just generate that Containerfile, then build the container with the tool of your choice.

Ansible Builder can build and tag the container for you.  If you navigate to the `execution_environment` directory in this repository, you'll be able to run the following command in that directory.  The command defaults to defining the container based on the file named `execution-environment.yml` in the directory where the command is run.
```
ansible-builder build --tag quay.io/hfenner/ee_mail
```

It's a good idea to store this image in a registry.  If you are running Ansible Automation Platform, the Private Automation Hub has a built-in registry.  Otherwise, any registry (such as Quay.io) works fine.

#### Ansible Navigator - https://ansible-navigator.readthedocs.io/en/latest/installation/
Ansible Navigator leverages the container you built to automatically import and run your playbook.  This can also be installed via pip:
```
pip install ansible-navigator
```

Ansible Navigator has a number of tools and options, but to get the experience closest to running `ansible-playbook` locally, try running:
```
ansible-navigator run 1-playbook/playbook.yml --eei quay.io/hfenner/ee_mail --mode stdout --penv EMAIL_PASSWORD
```

A couple of the options:
`-eei`:  Ansible Navigator has a default container that it will run if you don't specify the container you built via this flag.
`--mode`: Ansible Navigator defaults to an interactive mode that shows you a high level overview of task completion status.  This can be really useful if you are targeting a large number of hosts, but often you simply want to see streaming output.  This flag enables that
`-penv`: The container will not share the host environment values.  We're leveraging an environment variable to provide the password, so we have to explicitly tell Navigator to pass that value in.  A companion flag is `-senv` which will set rather than pass the environment variable.

#### Ansible Automation Platform (Red Hat supported version of AWX)
Using version control will help provide a more consistent experience for the logic in a playbook.  Using execution environments will help provide a more consistent runtime experience.  Unfortunately, you still have to have access to a system that can run containers and understand which flags to use for each situation.  Additionally, you will often need a secret such as a password or SSH key which you may not have access to.  Ansible Automation Platform (AAP) helps bring this all together by providing a GUI, a CLI, and an API that allow you to stage a playbook with the right inventory, playbook version, flags, execution environments, and credentials available to it called a job template.  In older versions of this software called Ansible Tower, this may have required some additional system level modification to install the right libaries and packages on the Ansible Tower system itself.  Execution environments solve this by allowing you to use the exact same runtime environment locally that you would use in Ansible.  Additionally, you can constrain a job template such that someone is able to run it, but not able to modify it in any way.

## Lab 2
Now that you've seen a simple playbook run, it's time to explore how the same playbook logic can be used to target different environments with different requirements.

The major difference in this lab is our variables are being sourced from inventory files as opposed to the playbook itself. This makes targeting multiple environments simpler.  You could use an extra_vars file instead, but inventories capture not just variables, but allow you to easily scope those variables by target.

First, let's remove the variables in the playbook as they will override variables in your inventory.  Variable precedence in Ansible is quite elaborate and has 22 levels of precedence (https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#ansible-variable-precedence).  General best practice is to define the variable once rather than relying on precedence.  In the case of roles, it's best practice to define variables in the role defaults directory as this will always be overridden by a variable declared anywhere else and allows the role to be consumed without having to define every option.

Our playbook looks like this now:
```
---
- hosts: build

  tasks:
  - name: Sending an e-mail using AWS Simple Email Service servers
    community.general.mail:
      host: "{{ host }}"
      port: 587
      username: "{{ username }}"
      password: "{{ password }}"
      from: "{{ email_source }}"
      to: "{{ email_dest }}"
      subject: Ansible-report
      body: "System {{ ansible_hostname }} has been successfully provisioned."
```

Our variables do need to be defined somewhere, so let's create an inventory:

migrated_vars_inventory.yaml
```
all:
  children:
    build:
      hosts:
        localhost:
      vars:
        ansible_connection: local
        host: email-smtp.us-east-2.amazonaws.com
        username: AKIAQY43FAYYIBN6X52W
        password: "{{ lookup('env', 'EMAIL_PASSWORD') }}"
        email_source: test@dragonslair.dev
        email_dest:
        - personone@dragonslayer.dev
        - persontwo@dragonslayer.dev
        - dragonslairpark@gmail.com
        
        email_dest: "{{ lookup('ansible.builtin.file', 'test.csv') | community.general.from_csv | map(attribute='email') }}"
```
Let's go ahead and run this again, but specify our newly create variable.  We should get the exact same outcome as the first lab.

```
ansible-navigator run 2-playbook/playbook.yml -i migrated_vars_inventory.yaml --eei quay.io/hfenner/ee_mail --mode stdout --penv EMAIL_PASSWORD
```

Static variables are simple which greatly aids readability and maintainability.  It can also hinder maintainability or cause duplication of data.  For example, you might want to notify a list of people that regularly changes.  Perhaps you have those emails saved in a spreadsheet, and want to be able to email those folks.  One quick option is to export that spreadsheet as a CSV.  We've included test.csv with three users to demonstrate this.

First, let's pull that CSV into Ansible.  We'll use a lookup for this.  We could also use a module, but a lookup is more flexible because we can quickly use ad-hoc commands to filter the data to just a list of e-mails, and we don't need to change our actual playbook logic at all.

```
ansible localhost -m debug -a msg="{{ lookup('ansible.builtin.file', 'test.csv') }}"
localhost | SUCCESS => {
    "msg": "first_name,last_name,email\nJohn,Doe,johndoe@dragonslair.dev\nJack,McGinnis,jackmcginnis@dragonslair.dev\nDragonsLair,Park,dragonslairpark@gmail.com"
}
```

That's cool, but we need a list of e-mail addresses so we will have to further parse this data.  Ansible leverages Jinja2 with some additional filters in order to reshape data.  Before we start, let's create a second inventory.  This makes it a little simpler parse a data structure especially when working with nested quotes (most shells make nested quotes difficult).  We're also going to rename the host because this inventory will target the same host with different variables as a demonstration.  If you name the host the same way in each group, you may find that variable precedence causes you headaches where the first group variables override other groups.

inventory_file_lookup.yaml
```
all:
  children:
    build_stage_1:
      hosts:
        stage1:
          ansible_host: localhost
          ansible_connection: local
      vars:
        ansible_connection: local
        host: email-smtp.us-east-2.amazonaws.com
        username: AKIAQY43FAYYIBN6X52W
        password: "{{ lookup('env', 'EMAIL_PASSWORD') }}"
        email_source: test@dragonslair.dev
        email_dest: "{{ lookup('ansible.builtin.file', 'test.csv') }}"
```

We can get the same lookup info by running our Ansible ad-hoc command as before, but add our new inventory with 
```
ansible -i inventory_file_lookup.yaml build_stage_1 -m debug -a msg="{{ email_dest }}"
```
```
stage1 | SUCCESS => {
    "msg": "first_name,last_name,email\nJohn,Doe,johndoe@dragonslair.dev\nJack,McGinnis,jackmcginnis@dragonslair.dev\nDragonsLair,Park,dragonslairpark@gmail.com"
}
```

So now we can pull in the information we need, but it's not yet in the format we need.  Let's try adding a Jinja2 filter.  If you've ever used bash pipelines before, this will feel very similar.  You use the vertical pipe to pass the data from the lookup to a filter that can break CSV into a list of dictionaries.
```
ansible -i inventory_file_lookup.yaml build_stage_1 -m debug -a msg="{{ email_dest | community.general.from_csv }}"
```
Output:
```
stage1 | SUCCESS => {
    "msg": [
        {
            "email": "johndoe@dragonslair.dev",
            "first_name": "John",
            "last_name": "Doe"
        },
        {
            "email": "jackmcginnis@dragonslair.dev",
            "first_name": "Jack",
            "last_name": "McGinnis"
        },
        {
            "email": "dragonslairpark@gmail.com",
            "first_name": "DragonsLair",
            "last_name": "Park"
        }
    ]
}
```

