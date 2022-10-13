# ISE CLI with Ansible

Ansible playbooks for ISE command line interface (CLI) operations.

## ISE CLI with Ansible

This will show you how to quickly install and use Ansible to run CLI commands against ISE.  While this may seem like a lot of work to run a few CLI commands that you could easily do with SSH, the ability to use these in a larger provisioning or automation workflow is very powerful!  The examples below will show you how work with CLI commands in exec mode, config mode, and interactive commands, and long-running commands and process monitoring via CLI.

## Setup

### GitHub Repository

All of the examples given here are conveniently available in a GitHub repository for you. You may clone this repository to your local computer using `git` :

```sh
git clone https://github.com/1homas/ISE_CLI_with_Ansible.git
cd ISE_CLI_with_Ansible
```

If you do not have Git, you may install it using

- macOS: `brew install git`  
- Debian/Ubuntu: `apt install git`
- RHEL: `yum install git`
- Windows: Install Windows Server for Linux (WSL) and install with one of the Linux methods above

### Install Python and Ansible

Ansible is built using the Python programming language so it must be installed first. You will use PIP - "PIP Installs Python (PIP)" to install the `pipenv` virtual environment utility, Ansible, and other Python packages required for ISE with Ansible.
When developing software, scripts, and automations, it is highly recommended to use a *virtual environment* for the packages and dependencies you use to prevent creating any conflicts with your normal computer environment and reduce the potential for unknown dependencies. The `pipenv` virtual environment is used below but you may use `venv` or any other virtual environment utility you like.
Create your Python environment and install Ansible:  

```sh
pip install --upgrade pip
pipenv install --python 3.9                     # use Python 3.9 or later
pipenv install ciscoisesdk jmespath paramiko    # ISE, JSON Query, SSH/CLI packages
pipenv install ansible                          # Ansible packages
pipenv shell                                    # launch the virtual environment
```

> 💡 Installing Ansible with Python PIP will get you the latest version.
> ⚠ Installing Ansible using Linux packages (`sudo apt install ansible`) may result in a much older version of Ansible being installed.
> 💡 If you have any problems installing Python or Ansible, see [Installing Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).

### Environment Variables

Environmental variables are defined for the current shell and used to *pass information into processes* that are spawned from the shell.  It is generally considered a [security best practice](https://12factor.net/config) to use *environment variables* to securely send passwords, tokens, or other credentials to your scripts and automation processes without revealing them to anyone looking over your shoulder or logs of your terminal history.  
You may view all current environment variables with the `env` command and individual variables with `printenv`.

```sh
env
env | grep ISE
printenv ISE_REST_PASSWORD
```

> 💡 Environment variables traditionally are named in UPPERCASE while shell variables are lowercase

You may define your environment variables using the `export` command in your \*nix shell :

```sh
export ISE_INIT_PASSWORD=C1sco12345

export ISE_USERNAME=admin
export ISE_PASSWORD=ISEisC00L
export ISE_VERIFY=False
export ISE_DEBUG=False

export ISE_BACKUP_REPOSITORY=FTP
export ISE_BACKUP_USERNAME=backup
export ISE_BACKUP_PASSWORD=ISEisC00L
export ISE_BACKUP_ENCRYPTION_KEY=ISEisC00L
```

Alternatively, you may keep your environment variables in text files in a `.secrets` or similar folder in your home directory and use `source {filename}` to load environment variables from the files:

```sh
source ~/.secrets/ise.sh
source ~/.secrets/ise_repository.sh
```

In Ansible, you reference variables and evaluate function within double curly braces: `{{ variable_name }}`. For environment variables, you do this with a `lookup()` function where you specify that you want `env` information and the specific variable you want:

```sh
ansible_ssh_user:     "{{ lookup('env','ISE_SSH_USERNAME') }}"
ansible_ssh_password: "{{ lookup('env','ISE_SSH_PASSWORD') }}"
```

> 💡 Beginning in ISE 3.2, all ISE cloud instances have the default superadmin username of `iseadmin`, not `admin`. This means you may need to change your scripts if you upgrading to ISE 3.2 with cloud instances or have hybrid cloud deployments.

### ansible.cfg

The `ansible.cfg` configuration file provides Ansible with your [settings and preferences](https://docs.ansible.com/ansible/latest/reference_appendices/config.html) like which file to use for the inventory (`hosts.yaml`) and how to display any output to `stdout` in your current project.

Some ISE CLI commands - including `reload`, `application reset-passwd`, `show application status ise` or `application start ise` may take longer than the default 30 seconds to complete.  For these CLI commands, the settings `connect_timeout` and `command_timeout` must be overriden with a longer time or the command will timeout and fail otherwise.
> 💡 You may override the persistent connection settings on a per-task basis with the `ansible_command_timeout` and `ansible_connect_timeout` variables. You will see examples of this later.

```ini
[defaults]
enable_plugins     = auto, yaml, host_list  ; inventory plugins and order used
inventory          = hosts.yaml        ; Ansible host file(s) (/etc/ansible/hosts.yml)
forks              = 5                 ; Concurrent processes allowed for hosts
host_key_checking  = false             ; Avoid SSH `known_hosts` key checking
stdout_callback    = yaml              ; Ansible output format: {default|minimal|yaml}
callbacks_enabled  = ansible.posix.profile_tasks  ; Show task execution times
interpreter_python = auto_silent       ; Silence warning about Python environment
## log_path           = ./ansible.log   ; Log file. Default: /var/log/ansible.log

##💡 Increase timeouts to 60s for ISE CLI `show app status ise`
[persistent_connection]
connect_timeout    = 60                ; Idle connection timeeout. Default: 30s 
command_timeout    = 60                ; Per-command timeout. Default: 30s

[ssh_connection]
pipelining = False                     ; improves performance; conflicts with `become`
```

### hosts.yaml

We will use a simple, static inventory file in YAML format for these CLI playbook examples.
You will need to update the `ansible_host` IP address with the address of your ISE PAN node.
Some comments about the `hosts.yaml` inventory file:

- we are not creating any global variables here - look in `vars.yaml` instead
- the localhost is our Ansible control node so we tell it to consider it `local` and not use SSH by default
- we use a series of `ansible_*` variables for our `ise` node and the `ansible_connection` and the `ansible_network_os` are the most important for [Cisco IOS platform-specific CLI behavior](https://docs.ansible.com/ansible/latest/network/user_guide/platform_ios.html) with SSH
- we will connect with an SSH username and password by default but you could use an SSH key if configured

```yaml
all:
  vars:
    # no global vars
  hosts:
    localhost:
      ansible_connection: local # do not use SSH by default

    ise:
      ansible_host: 198.18.133.27
      ansible_become: no # ISE does not have a superuser/enable mode
      ansible_connection: ansible.netcommon.network_cli
      ansible_network_os: cisco.ios.ios
      # ⚠ ISE SSH username is `iseadmin` for ISE 3.2+ cloud instances
      ansible_ssh_user: "{{ lookup('env','ISE_SSH_USERNAME') }}"
      ansible_ssh_password: "{{ lookup('env','ISE_SSH_PASSWORD') }}"
      # ansible_ssh_private_key_file: "{{ ssh_private_key_file }}"
```

> 💡 Use variables for passwords instead of saving your actual passwords in your Ansible projects!

Double-check your environment variables using are correct:

```sh
env
env | grep ISE_
printenv ISE_SSH_USERNAME
printenv ISE_SSH_PASSWORD

### SSH Connectivity

You *must* have SSH connectivity to ISE using either an SSH username+password or username+ssh_key for Ansible to work with ISE CLI. It is recommended that you try it first to verify your credentials work before you attempt to use them with Ansible:
```sh
ssh {ise_admin}@ise.securitydemo.net
ssh -i ~/.ssh/{ssh_private_key_file} {ise_admin}@ise.securitydemo.net
```

## Show Commands

### show version

The easiest way to start using Ansible with ISE CLI is by doing the simplest show command, `show version`.

```yaml
---
- name: Show ISE Version
  hosts: ise
  gather_facts: no
  tasks:
    - name: ISE CLI | show version
      ansible.netcommon.cli_command:
        command: show version
      register: output

    - name: Show output
      ansible.builtin.debug:
        msg: "{{ output.stdout | replace('\n\n','\n') }}" # remove empty lines
```

This Ansible playbook will run the `show version command` using the `ansible.netcommon.cli_command` module then use the `ansible.builtin.debug` module to print it.  It will run on the inventory host `ise` as defined in your `hosts.yaml` file above. If you change it in the hosts file, you will want to change it in your playbook, too.

> 💡 This playbook uses the `ansible.netcommon.cli_command` instead of the `cisco.ios.ios_command` module because it allows us to strip out the empty lines in the output. For some reason, the `replace()` function does not work with the `ios_command` output.  If you figure out why, please let me know!

Run the show version

```sh
ansible-playbook show_version.yaml
```

You should see output similar to this:

```sh
❱ ansible-playbook show_version.yaml

PLAY [Show ISE Version] ********************************************************

TASK [ISE CLI | show version] **************************************************
[WARNING]: ansible-pylibssh not installed, falling back to paramiko
ok: [ise]

TASK [Show output] *************************************************************
ok: [ise] =>
  msg: |-
    Cisco Application Deployment Engine OS Release: 3.2
    ADE-OS Build Version: 3.2.0.401
    ADE-OS System Architecture: x86_64

    Copyright (c) 2005-2022 by Cisco Systems, Inc.
    All rights reserved.
    Hostname: ise


    Version information of installed applications
    ---------------------------------------------

    Cisco Identity Services Engine
    ---------------------------------------------
    Version      : 3.2.0.542
    Build Date   : Tue Aug 30 05:21:58 2022
    Install Date : Fri Sep 23 09:25:02 2022

PLAY RECAP *********************************************************************
ise  : ok=2  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0

```

That was easy!

### show running-config

Probably the next most familiar, non-destructive command to run is `show running-config`

```yaml
---
- name: Show ISE Configuration
  hosts: ise
  gather_facts: no
  tasks:
    - name: ISE CLI | show running-config
      ansible.netcommon.cli_command:
        command: show running-config
      register: output

    - name: Show output
      ansible.builtin.debug:
        msg: "{{ output.stdout | replace('\n\n','\n') }}" # remove empty lines
```

Run the show version

```sh
ansible-playbook show_running-config.yaml
```

And you should see your entire ISE node's configuration on the screen!

## Interactive Commands

Some executive commands may prompt your for information (`application reset-passwd`), confirmation, or both (`reload`) requiring your automation tasks to prompt you or anticipate the prompts.

### ISE Admin Password Reset

The normal ISE admin password reset process is this:

```sh
ise/admin#application reset-passwd ise admin
Enter new password:
Confirm new password:

Password reset successfully.
```

The `ansible.netcommon.cli_command` module has the ability to match prompts and supply an answer in response to them. This is perfect for when you wan to reset the admin password in ISE.

```yaml
---
- name: Show ISE Configuration
  hosts: ise
  gather_facts: no
  vars_prompt:
    - name: username
      prompt: What is the username to reset?
      default: admin
      private: no
    - name: new_password
      prompt: What is the new password?
      private: yes
  tasks:
    - name: Password Reset | {{ username }}@{{ inventory_hostname }}
      ansible.netcommon.cli_command:
        command: "application reset-passwd ise {{ username }}"
        check_all: yes
        sendonly: no
        newline: yes
        prompt:
          - "Enter new password:"
          - "Confirm new password:"
        answer:
          - "{{ new_password }}\r"
          - "{{ new_password }}\r"
      register: output
      failed_when: "'successfully' not in output.stdout"
```

```sh
❱ ansible-playbook password_reset.yaml
What is the username to reset? [admin]:
What is the new password?:

PLAY [Show ISE Configuration] **************************************************

TASK [Password Reset | admin@ise] **********************************************
ok: [ise]

PLAY RECAP *********************************************************************
ise  : ok=1  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

### reload

If you want to `reload` an ISE node, the command is very simple but the process takes *several minutes* which will lead to a timeout if you do not take special precautions with your Ansible task.  Here is the normal `reload` process:

```sh
ise/admin#reload
Save the current ADE-OS running configuration? (yes/no) [yes] ? yes
Generating configuration...
Saved the ADE-OS running configuration to startup successfully

Broadcast message from root@ise (pts/1) (Sun Oct  9 15:14:28 2022):

Trying to stop processes gracefully. Reload might take approximately 3 mins
```

Note the process clearly says it takes ***3 minutes*** which is much longer than the 60 seconds configured for the `command_timeout` setting in `ansible.cfg`!  To handle this long execution time for the `reload` command, we will increase the timeout by overriding it for this task with the local `ansible_command_timeout` variable.

```yaml
- name: ISE reload
  hosts: ise
  gather_facts: no
  tasks:
    - name: reload | {{ inventory_hostname }}
      vars:
        #  ⚠ ISE 3.2: A command timeout of < ~7 minutes will cause a failure!
        ansible_command_timeout: 600
      ansible.netcommon.cli_command:
        command: reload
        check_all: yes # run all of the commands
        prompt:
          # Save the current ADE-OS running configuration? (yes/no) [yes] ?
          - Save the current ADE-OS running configuration
          # Continue with reboot? [y/n]
          - Continue with reboot
          # Trying to stop processes gracefully. Reload might take approximately 3 mins
        answer:
          - "yes"
          - "y"
      register: output
      failed_when: "'gracefully' not in output.stdout"

```

## Monitor a Process

Now that you have *reloaded* your ISE node, you probably would like to monitor it to know when the ISE `Application Server` process is finally `running` again so that you may login to the GUI. The playbook `application_status_running.yaml` is for you!

This playbook will wait for up to 10 minutes to be sure the SSH port is available in the case of a full reboot. It will then continue to run the `show application status ise` command until it gets a regex match on the Application Server line with the word `running` then it will print an [ASCII art banner](http://patorjk.com/software/taag/#p=display&f=Standard&t=Ready!) showing ISE is Ready!

```yaml
---
- name: Monitor ISE Application Status
  hosts: ise
  gather_facts: no
  tasks:

    - name: Wait for SSH | {{ inventory_hostname }}
      ansible.builtin.wait_for:
        host: "{{ ansible_host }}"
        port: 22          # SSH
        sleep: 10         # Default: 1.   Seconds to sleep between checks
        timeout: 600      # Default: 300. Stop checking after <seconds>


    # 💡 Application Server STATE may be {not running|initializing|running}
    #--------------------------------------------------------------------------
    # Example output:
    #
    # ISE PROCESS NAME                       STATE            PROCESS ID
    # ...
    # Application Server                     running          55108
    # ...
    #--------------------------------------------------------------------------

    - name: Wait for ISE Application Server 'running' | {{ inventory_hostname }}
      vars:
        ansible_command_timeout: 90 # 💡 command requires ~40+ seconds
      ansible.netcommon.cli_command:
        command: show application status ise
      register: output
      until: output.stdout is regex('Application Server[ ]+running')
      retries: 20


    # 💡 Any empty lines in the pause prompt will be trimmed by Ansible!
    #    The empty lines below begin with a Unicode U+2003 Em Space [ ]!
    - name: ISE Login Ready
      ansible.builtin.pause:
        seconds: 0
        prompt: |
           
           
                     .
                    /|\          ____                _       _
                @  /|||\  @     |  _ \ ___  __ _  __| |_   _| |
               @  /|||||\  @    | |_) / _ \/ _` |/ _` | | | | |
               @  \|/ \|/  @    |  _ <  __/ (_| | (_| | |_| |_|
                @.       .@     |_| \_\___|\__,_|\__,_|\__, (_)
                 `Y@ @ @Y`                             |___/
           
           
```

## Configuration

### Enable DNS caching

The Cisco TAC Lead for ISE tells me a common problem that customers have is latency introduced with syslog servers configured with DNS names because every call to the syslog server generates a DNS request if caching is not enabled!  We are going to solve that be turning on DNS caching in ISE.  This would be the normal command sequence :

```text
ise/admin#configure terminal
Entering configuration mode terminal
ise/admin(config)#service cache enable hosts ttl 300
Successfully enabled DNS cache
```

The equivalent Ansible task for this uses the `cisco.ios.ios_config` module and we include a task we used earlier to dump out the ISE configuration to see the result in `configure_service_cache_enabled.yaml` :

```sh
---
- name: Configure DNS Service Cache
  hosts: ise
  gather_facts: no
  tasks:
    - name: ISE CLI | Enable DNS Caching | {{ inventory_hostname }}
      cisco.ios.ios_config:
        lines: service cache enable hosts ttl 300
      register: output
      failed_when: output.failed

    - name: ISE CLI | show running-config
      ansible.netcommon.cli_command:
        command: show running-config
      register: output

    - name: Show output
      ansible.builtin.debug:
        msg: "{{ output.stdout | replace('\n\n','\n') }}" # remove empty lines
```

## License

This repository is licensed under the [MIT License](https://choosealicense.com/licenses/mit/).
