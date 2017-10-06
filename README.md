# Ansible role: Backup your hosts with [Borg Backup](http://borgbackup.readthedocs.io/en/stable/index.html).

An ansible role that installs Borg Backup on RHEL/CentOS 7

## Requirements

This role uses two other roles: [ansible_collect_ssh_keys](https://github.com/mikhail-github/ansible_collect_ssh_keys) for generate ssh keys and [ansible_authorize_ssh_keys](https://github.com/mikhail-github/ansible_authorize_ssh_keys) for authorize those public keys on backup server.
Borg Backup can be found in EPEL repo, so EPEL should be installed before.
Backup server should have installed [Borg Backup](http://borgbackup.readthedocs.io/en/stable/installation.html).

## Role variables

User for run backup crontab jobs:

	backup_client_user: "root"

Backup server FQDN or ip:

	backup_server: "backupserver.local"

User on backup server, should have write permissions to a repository directory.

	backup_server_user: "borg"

Path for create a repository for a host:

	backup_repos_path

Or define one repository for all hosts:
	
	#backup_repository: "general_repo"

Email notification settings:

	backup_emailto: "ops@dom.local"
	backup_emailfrom: "noreply@dom.local"

If you want to remove all crontab jobs with word borg before adding a new ones - set this to true

	remove_crontab_jobs: false

Your backup server public key, can be found by command ssh-keyscan -t rsa backupserver:

	backup_server_key: ""

## You can find some other defaults in role/defaults/main.yml

Default list of directories for all hosts:

	borg_default_backup_dirs:
          - "/etc"

Default values for backup schedulle:

	borg_schedulle_defaults: { minute: "00", hour: "00", wday: "*", day: "*", month: "*" }

Default job command line:

	borg_default_cmd: "{{ borgwrapper }} -c /etc/borgbackup.conf allops >> /var/log/borgbackup.log 2>&1"

Shell script path:

	borgwrapper: "/usr/sbin/borgwrapper"

## You can define hosts specific variables in borg variable

	borg[host_fqdn]

Example vars/main.yml:

	borg:
	  host1.local:
	    schedulle:
	      - { minute: "00", hour: "01", name: "myhost01"}
	      - { minute: "30", hour: "19", name: "myhost19"}
	    backup_dirs:
	      - "/data"
	      - "/data2"
	  host2.local:
	    schedulle:
	      - { minute: "45", hour: "*", name: "myhost21"}
	    backup_dirs:
	      - "/tmp"
	    cmd: "{{ borgwrapper }} -c /etc/borgbackup.conf backup"

## Example playbook

You can found it in sample-playbook.yml

- hosts: borgbackup
  roles:
    - role: collect_ssh_keys
      #the user for backup jobs running, should be the same as backup_client_user
      ssh_user: "root"

- hosts: backupserver
  roles:
    - role: authorize_ssh_keys
      ssh_keys: "{{ groups['borgbackup'] | map('extract', hostvars, ['ssh_public_key']) | list }}"
      #the user for communicate with backup server, should be the same as backup_server_user
      ssh_user: "borg"
      ssh_key_options: 'command="borg serve --restrict-to-path /borg/repos",no-pty,no-agent-forwarding,no-port-forwarding,no-X11-forwarding,no-user-rc'

- hosts: borgbackup
  roles:
    - role: setup_borg_client
      backup_client_user: "root"
      backup_server: "backupserver.dom.local"
      backup_server_user: "borg"
      backup_repos_path: "/borg/repos"
      #backup_repository: "general_repo"          # define if you want one repo for hosts
      backup_emailto: "ops@dom.local"             # notify recipient
      backup_emailfrom: "noreply@dom.local"       # from address
      remove_crontab_jobs: false                  # remove all borg jobs before adding new
      backup_server_key: ""                       # can be found by ssh-keyscan -t rsa backupserver

