# Pokemon Unite Setup

## Hosts
- There are multiple hosts:
  - the windows hosts are the gameservers
  - the linux hosts is the monitoring and ansible endpoint for the windows hosts

### Windows
For windows you need to run the win-req.ps1 file: 
`powershell.exe -executionpolicy bypass win-req.ps1`

Make sure the user you put in the inventory file (production.yml) has a password enabled. 
TODO: test windows ssh, should be able to use key based auth: https://docs.ansible.com/ansible/latest/os_guide/windows_ssh.html

### Linux
For linux make sure you have a user with key based auth enabled. 

## Ansible Setup
1. Create a python virtual environment:
`python3 -m venv ~/ansiblevenv`
2. Activate the ansible venv: (you need to do this everytime to execute playbooks)
`source ~/ansiblevenv/bin/activate`
3. Install the ansible python requirements:
`pip3 install -r requirements.txt`
4. create a vault_pass.sh file containing the vault pass to decrypt the encrypted variables (such as the docker login). The file can contain the password cleartext or be a script that returns the password (such as 1password-cli). 

## Running
### Linux
Run the common role: 
`ansible-playbook -i production playboosk/common.yml -u <user>`
This will setup:
- hostname, ntp, users, packages, dns resolve config, node exporter and networking with ifupdown2. 

Run the docker role and the monitoring role. 
- Docker will setup docker on linux, install portainer as well
- The monitoring role will install prometheus, blackbox exporter and grafana. Setup of grafana is done on the go, no config is maintained atm. 

### Windows
Run the common role, this will: 
- Set hostname, disable standby and screentimeout 
- install windows exporter and open the firewall for it
- install chocolatey and wsl2 with chocolatey (this is the only way that always worked)
- install docker desktop and make sure it starts on boot. 

Run the unite role:
- it will template a .bat file on the desktop to run the servers
- set windows firewall for unite
- run docker login with the encrypted variables
- pull the required docker container already. 

## Unite Specific
After the unite role has been ran, run the .bat file on the windows desktop manually. 

### Server id
There are variables under production.yml (but they can be under host_vars as well if wanted) containing: 
```
unite_server_id: 7
unite_log_id: 303
```
The `unite_server_id` variable will be templated into the docker image "honolulu" id. 
the `unite_log_id` variable will be used to create the bind mount for the logs. 


### Whitelist
when the server is running support ids can be whitelisted in the server. 
Create the following example variable per host:
```
whitelist_users:
  - id: 1065-7744-5551-6843-5946
      comment: 092 - Mainstage - Main
  - id: 1479-1285-5713-3152-3648
      comment: 096 - Mainstage - Backup
```

WARNING: ONLY WHITELIST A SUPPORT ID ON 1 SERVER. 
When whitelisting it seems that the cloud tells the client to which server to connect locally, if you have whitelisted on multiple local servers it might push another local server to the whitelisted client. 

TODO: the above could be interesting to test L3 servers, L2 is no requirement, we had L3 suddenly working on EUIC2025. As long as the clients are able to reach the "inner_ip" configured on the server it should work. 

After you have read the above carefully, run: 
`ansible-playbook -f 10 -i production.yml playbooks/unite.yml --tags whitelist`

### Getting the player numbers on one device
- enable unsecure docker host expose on the docker desktop settings
- the port forward and firewall rule is being done in the docker-desktop tasks

There is a folder called "envdocker" these have the environment for each server to connect to, source the file: `source envdocker/sv<id>` and now run the docker command to get the player numbers but include `watch -n 1` to run this command each second. 
`watch -n 1 docker exec -it -u root localsvr /bin/sh /data/home/user00/ieg_ci/rd_script/monitor.sh`

### Future
- See if we can forward the logs to a Loki instance for all the servers
- The server seems to require minimal resources, it should be easy to run 50+ servers on 1 pc or a server cluster instead of 1 server per laptop
Minimal resources: a bump of 1% cpu on the servers, about 1/1mbit (ingame, lower in lobby) for 11-14 clients per server. Memory usage also not noticable. Images are quite big at 8G a piece
- There is a linux poc in the pkm-unite folder, as the only requirement is docker it should run way more stable on Linux, no weird windows update stuff and wsl required. 
- If linux would be used we can maybe use cadvisor to monitor the container usage specifically. 
- Reasons why there are differente docker images per server, what is the difference, can we pass this to the command line or as a config file instead of a different 8G image per server. 
- Monitor script to prometheus scrapable endpoint, also return the game phase and some client details?
