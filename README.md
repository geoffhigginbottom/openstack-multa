# Deploy OpenStack onto Rackspace Public Cloud using OpenStack-Ansible

This will deploy a fully working version of OpenStack on top of the Rackspace
Public Cloud using OpenStack-Ansible.

In its current form it will deploy the following:

    3x Controllers
    3x Compute
    3x Swift
    1x Cinder
    1x HA Proxy LB
    1x Gateway
    1x Console


## Ansible Vault

A number of sensitive setttings are required to be stored within the configuration of this repo.  To protect this sensitive data we use Ansible Vault.  A "vault\_example.yml" file can be used to generate your own vault.yml file within "group\_vars/general".

The "vault\_my\_cloud\_password" variable is used when setting up a user account once OpenStack has been deployed.  As this system may be accessible from the Internet, it is important that a strong password is used.

The various mail settings are used to allow the sending of e-mails during the deployment, this behaviour can be disabled (see Mail Alerting settings below)

Example vault.yml file with settings for using an apple mail account for sending the mail alerts.

	---
	
	# Configure Openstack Settings
	vault_my_cloud_password: xxxxx
	
	# Status Mail Settings
	vault_host: smtp.mail.me.com
	vault_port: 587
	vault_secure: starttls
	vault_username: xxx@apple.com
	vault_password: xxxxxxxx
	vault_from: xxxxx@me.com
	vault_to: First Last <xxxx@me.com>

Once you have created the vault.yml file you shuld encrypt it using the following command:

	ansible-vault encrypt group_vars/general/vault.yml

When using Ansible Vault, you can either pass the password used to encrypt the vault file at run time by using --ask-vault-pass, or you can generate a .vault_pass file in the root of the repo and update the ansible.cfg file to include "vault\_password\_file = ./.vault\_pass"

Note this repo assumes there is a .vault_pass file and the ansible.cfg is configured accordingly

More info on using Ansible Vault is available [here] (http://docs.ansible.com/ansible/latest/user_guide/vault.html)

## clouds.yaml

You will need to create a clouds.yaml to configure the authentication to your base cloud which you will be using to deploy your test cloud.  There is a clouds_example.yaml included which you can use as a template; Note that .gitignore is configued to not sync this back to github (vault cannot be used for clouds.yaml settings)

Example clouds.yaml file

	clouds:
	
	  xxxx:
	    auth:
	      auth_url: https://identity.api.rackspacecloud.com/v2.0
	      username: xxxx
	      password: xxxx
	      project_name: nnnnnn
	    region_name: XXX
	
	
	  IAD:
	    auth:
	      auth_url: https://identity.api.rackspacecloud.com/v2.0
	      username: xxxx
	      password: xxxx
	      project_name: nnnnnn
	    region_name: IAD
	
	
	ansible:
	  use_hostnames: True
	  expand_hostvars: False
	  fail_on_errors: True

I assume you will have multiple accounts in clouds.yaml so you will need to
populate the OS_CLOUD environment variable with the cloud you want to work
against. 

e.g. 

	export OS_CLOUD=IAD

or 

	export OS_CLOUD=DFW

However if you only have a single active entry in clouds.yaml, or you comment out inactive accounts, then there is no need to export the OS_CLOUD variable.

## Steps for deployment:

1. Clone the repo
2. Review settings in /group\_vars/general and amend as required
3. Generate "group\_vars/general/vault.yml" using vault\_example.yml as a template
4. Generate a '.vault\_pass' file or comment out the "vault\_password\_file" setting in ansible.cfg and provide the decrypt password manually at run time
5. Generate clouds.yaml
6. Deploy base VMs using the Heat template openstack-osa-framework.yaml, updating parameters as required - Ensure you update the key\_name setting to match the name of your ssh key
7. Update the console\_user\_pwd setting in host\_vars/console.yml - this is the hashed password for the local user on the console VM
8. Deploy OpenStack by running the below command

~~~
ansible-playbook site.yml
~~~
    
## Typical Run Times

    Initial VM Deployment using Heat: 7 mins
    VM Preparation Playbooks: 11 mins
    setup-hosts.yml: 49 mins
    setup-infrastructure.yml: 55 mins
    setup-openstack.yml: 58 mins
    console-vm.yml: 12 mins
    Total: approx 3.5 hours


## Obtain the Admin Password

Openstack-Ansible generates strong random passwords for all the different services and you will need to ascertain the admin password if you want to log into the Horizon UI.  This can be obtained in number of ways:

Connect to the deployment server, controller1 via SSH and run the following command:

~~~
grep keystone_auth_admin_password /etc/openstack_deploy/user_secrets.yml
~~~

Alternatively, from your host where you initially ran this deployment from, run the following playbook

~~~
ansible-playbook admin_pwd.yml
~~~

Another option is to connect to the Console VM via SSH and run the following command

~~~
grep OS_PASSWORD /ansible/openrc | cut -d= -f2
~~~

There is also a "utility\_container" on each of the controller nodes which also has an openrc file located in /root/openrc so just like on the console VM you can obtain the admin password by running the following

~~~
grep OS_PASSWORD /root/openrc | cut -d= -f2
~~~


## Connecting to the Horizon UI

To connect to the Horizon UI, simply connect to the Public IP of the LB1 VM using https://nn.nn.nn.nn  As we are using a self signed certificate you will get a warning from your browser, ensure you are connecting to the correct IP then accept the warning and you should be presented with the login screen.  Login as admin, using the password you obtained using any of the above methods.

## VPN Access To Provider Network
You can use the Console VM to test SSH access to VMs or use the Desktop Browser to test HTTP access etc, but if you want to do this directly from your machine, the easiest thing to do is install a VPN Server on the GW VM.

Simply install by downloading latest package from [this page] (https://openvpn.net/index.php/access-server/download-openvpn-as-sw/113.html?osfamily=Ubuntu) e.g.


    wget http://swupdate.openvpn.org/as/openvpn-as-2.5.2-Ubuntu16.amd_64.deb

Then install it by running 

    dpkg -i openvpn-as-2.5.2-Ubuntu16.amd_64.deb 
    
Once installed, set a password for the openvpn user

    passwd openvpn

Now connect to the Public IP of the GW VM and use the UI to configure the VPN, logging in using openvpn as the username, and the password you configured in the previous step

    https://nn.nn.nn.nn/admin

The following settings should be changed:

1. Configuration/VPN Settings/Routing - replace the existing CIDRs with the 'Flat Network CIDR which is "172.29.252.0/22" unless you have changed it
2. Configuration/VPN Settings/Routing - 'Should client Internet traffic be routed through the VPN?' - Set to "No"
3. Configuration/VPN Settings/DNS Settings - 'Do not alter clients' DNS server settings' - Set to "Yes"

Click the "Save Settings" button at the bottom of the screen, then the "Update Running Server" button which appears at the top of the screen once the settings are saved.

You can now connect to the VPN by installing the appropriate openvpn client and downloading the connection profile directly from the server.  I opt to create a new 'user' which has a descriptive name for the environment I am connecting to.

(A future version of this repo will automate the installation and configuration of the VPN)

## Mail Alerting

As the deployment takes approximately 3.5 hours from start to finish, it can be useful to monitor its progress via e-mail.  To disable this you need to set "send\_mail: false" in the /group_vars/general/vars.yml file (it is enabled by dafult).  You will also need to complete all the "Status Mail Settings" in the vault.yml file. 

## Troubleshooting

The initial steps can be monitored from your local host machine, but once it gets to the osa-deploy.yml where Ansible on the deployment host takes over to run the setup-hosts.yml, setup-infrastructure.yml and setup-openstack.yml playbooks, you will need to connect to the  deployment host if you want to monitor their progress.

Run the following command on the deployment host to monitor the deployment progress:  

    tail -f /openstack/log/ansible-logging/ansible.log 
    
It is normal for the deployment to appear to hang on "TASK [repo_build : Create OpenStack-Ansible requirement wheels]" just be patient and allow it time to complete.

If you have problems with failures when running "ansible-playbook site.yml" comment out the steps which run "setup-hosts.yml, setup-infrastructure.yml and setup-openstack.yml" at the end of the osa-prep.yml playbook, and also the console-vm.yml in site.yml.  Run ansible-playbook site.yml but then when it finishes, connect to the deployment host and run the above steps one by one, and trouble shoot any issues found.

If you have problems with the importing of images ensure Swift is functioning as Glance is using this as its backend.  On occassion I have had success by simply re-running the os-swift-install.yml playbook directly from the deployment host.  The install playboks can be found in /opt/openstack-ansible/playbooks.

