# Setting up JupyterHub for Multi-User Support on RHEL8 with a non-root user

This a quick guide for setting up JupyterHub with multi-user support on Red Hat Enterprise Linux 8.
To start, log in as the root user on the machine that will run the server and run the following commands to install the pre-requisites: 
```
# yum install git
# yum module install python38
# pip3 install - upgrade setuptools
# yum module install nodejs:14
# pip3 install jupyter
# pip3 install jupyterhub
# pip3 install git+https://github.com/jupyter/sudospawner
# pip3 install jupyterlab
```
Now we add the jupyterhub user that will be used to run the service on startup:
```
# useradd jupyterhub -r
```
Update the sudoers file:
```
# visudo 
```
Add the following to end of the line for secure_path (colon separated): /usr/local/bin. If this step is skipped, jupyterhub will have permission errors/issues when users try to login.
Next, make a directory where jupyterhub will be launched from and generate the jupyterhub config:
```
# mkdir /etc/jupyterhub
# chown jupyterhub /etc/jupyterhub
# cd /etc/jupyterhub
# /usr/local/bin/jupyterhub - generate-config
```
I added the following line to the jupyterhub_config.py file (from https://github.com/jupyterhub/jupyterhub/issues/930#issuecomment-444049046):
```
c.ConfigurableHTTPProxy.command = '/usr/local/bin/configurable-http-proxy'
```
Now install the configurable-http-proxy with npm:
```
# npm install -g configurable-http-proxy
```
Now the jupyterhub executable can be launched as root without any issues on port 8000 (the default). You can test at your localhost with your web browser at http://localhost:8000. Though you may not be able to log in yet.
After doing this, and adding the following to the sudoers file or create a custom file with (Source: https://github.com/jupyterhub/jupyterhub/wiki/Using-sudo-to-run-JupyterHub-without-root-privileges):
```
# visudo /etc/sudoers.d/jupyterhub
```
I was able to log into jupyterhub with my Active Directory Account (The user list is pulled from elsewhere in my environment):
```
# You may want to add a comma-separated whitelist of users here

# Runas_Alias JUPYTER_USERS = example, user
# the exact path may differ, depending on how sudospawner was installed
cmnd_Alias JUPYTER_CMD = /usr/local/bin/sudospawner, /usr/sbin/mkhomedir_helper, /usr/libexec/oddjob/mkhomedir
# actually give the Hub user permission to run the above command on behalf
# of the above users without prompting for a password
jupyterhub ALL=(ALL) NOPASSWD:JUPYTER_CMD
```
At this point, I proceed to follow the guide and set up the jupyterhub user that can launch without being root: https://jupyterhub.readthedocs.io/en/stable/reference/config-sudo.html
Ran the following command to verify help options show up (it did):
```
# sudo -u jupyterhub sudo -n -u $USER /usr/local/bin/sudospawner - help
```
And the following command is supposed to prompt a password (it did):
```
# sudo -u jupyterhub sudo -n -u $USER echo 'fail'
```
**Optional** (for those trying to setup local users on the machine to login):
You can add a shadow group so PAM can read the shadow password database:
```
# groupadd shadow
# chgrp shadow /etc/shadow
# chmod g+r /etc/shadow
# usermod -a -G shadow jupyterhub
```
Then run this command to determine the changes took effect: 
```
# ls -l /etc/shadow
```
Verify PAM is working with: 
```
# sudo -u jupyterhub python3 -c "import pamela, getpass; print(pamela.authenticate('$USER', getpass.getpass()))"
```
In our case, since we're AD-joined with sssd, we don't need to make changes to PAM or any local files like /etc/group. We simply give users the ability to login with AD credentials based on AD group memberships. E.g. 
```
# realm permit –g my-ad-groupname
```
Now we try to start the server as the jupyterhub user:
```
# sudo -u jupyterhub /usr/local/bin/jupyterhub - JupyterHub.spawner_class=sudospawner.SudoSpawner
```
Alternatively, you could just switch to the jupyterhub user as root (su jupyterhub) and then remove the "sudo -u jupyterhub" at the start of the above command. If you get an error about permission being denied for an invalid cookie secret file or an attempt to write to a read only database simply shut the server down and delete the jupyterhub_cookie_secret and jupyterhub.sqlite files in the /etc/jupyterhub directory.
For the next step we want to set JupyterLab as the default/home page when a user logs into jupyterhub. To accomplish, I modified this following section of the jupyterhub_config.py file:
```
c.Spawner.default_url = '/lab'
```

Restart jupyterhub and verify that upon logging into the service, you are redirected to the /lab URL. 
The next step was to get this script to launch on startup with systemd using the following guide as reference: https://github.com/jupyterhub/jupyterhub/wiki/Run-jupyterhub-as-a-system-service
We will create a unit file to run systemd service as specific user:
```
[Unit]
Description=Run service as user jupyteruser
[Service]
User=jupyterhub
ExecStart=/usr/local/bin/jupyterhub - JupyterHub.spawner_class=sudospawner.SudoSpawner
WorkingDirectory=/etc/jupyterhub
[Install]
WantedBy=multi-user.target
```
I named the above script jupyterhub.service and placed it in /lib/systemd/system/jupyterhub.service. Alternatively, it could also be placed in /etc/systemd/system.
```
# cd /lib/systemd/system/
# vi jupyterhub.service
```
Then paste the above into this file and save it with vi. Refresh the systemd configuration files and enable the service to start automatically at boot:
```
# systemctl daemon-reload
# systemctl enable jupyterhub.service
```
Test restarting systemd with the service:
```
# systemctl restart jupyterhub.service
```
Check the status:
```
# systemctl status jupyterhub.service
```
You may need to delete the jupyter_cookie_secret and jupyterhub.sqlite files in the /etc/jupyterhub directory and reboot the machine to verify that this change works on start up.
Setting up HTTPS with Jupyterhub
Next I wanted to see if I could get HTTPS working before limiting the jupyterhub user to non-interactive logins. Generate a SSL key and certificate based on your local policy and install at the path shown in the jupyterhub_config.py file. I used a self-signed certificate for initial setup and testing. Then the following lines must be added or changed accordingly in the jupyterhub_config.py:
```
c.JupyterHub.ssl_cert = '/etc/jupyterhub/server.crt'
c.JupyterHub.ssl_key = '/etc/jupyterhub/server.key'
```
Then I ran the following commands from the /etc/jupyterhub directory to make sure that the jupyterhub user has permission to read/access the cert and key upon execution:
```
# chown jupyterhub:jupyterhub server.crt
# chown jupyterhub:jupyterhub server.key
# chmod 0400 server.key
```
Restart the jupyterhub service to test the new settings. The server was now working with HTTPS, however I received a warning about the certificate being invalid. This is expected since I self-generated the certificate for this experiment. This cert can be changed to a verified/signed cert in final implementation.
Restricting login access on the jupyterhub user
Next I wanted to restrict login access for the jupyterhub user on my server.
A good reference: https://www.tecmint.com/block-or-disable-normal-user-logins-in-linux/
I ran the following command:
```
# chsh -s /sbin/nologin jupyterhub
```
I was able to confirm this worked when trying to switch to the jupyerhub (su jupyterhub).
Setting up Jupyterhub with nginx
Finally, I want to make the service accessible on standard web ports 80 and 443. To do this, I use a simple nginx setup to redirect these ports to https://fqdn:4443.
Install nginx:
```
# yum install nginx
```
Then I referenced the following guides to proceed:
https://jupyterhub.readthedocs.io/en/stable/reference/config-proxy.html
https://github.com/jupyterhub/jupyterhub-the-hard-way/blob/master/docs/installation-guide-hard.md
I came across an issue and associated bug report (https://stackoverflow.com/questions/42078674/nginx-service-failed-to-read-pid-from-file-run-nginx-pid-invalid-argument). Apparently it is caused by a race between nginx and systemd, which does not surprise since I set up jupyterhub on autoboot.
Work around:
```
# mkdir /etc/systemd/system/nginx.service.d
# printf "[Service]\nExecStartPost=/bin/sleep 0.1\n" > /etc/systemd/system/nginx.service.d/override.conf
```
Next we modify the nginx configuration file: 
```
vi /etc/nginx/nginx.conf
```
Inside the block for http, the following line was added below "server_name _;" for port forwarding:
```
return 301 https://$host:4443;
```
The following lines below "Settings for a TLS enabled server" get changed from their default:
```
# Settings for a TLS enabled server.
#
server {
listen 443 ssl http2 default_server;
listen [::]:443 ssl http2 default_server;
server_name _;
# redirect all traffic to https for node.js frontend of JupyterHub on port 4443
return 301 https://$host:4443;
root /usr/share/nginx/html;
#
ssl_certificate "/etc/pki/tls/certs/server.crt";
ssl_certificate_key "/etc/pki/tls/private/server.key";
ssl_session_cache shared:SSL:1m;
ssl_session_timeout 10m;
ssl_ciphers PROFILE=SYSTEM;
ssl_prefer_server_ciphers on;
#
# # Load configuration files for the default server block.
include /etc/nginx/default.d/*.conf;
location / {
}
error_page 404 /404.html;
location = /40x.html {
}
error_page 500 502 503 504 /50x.html;
location = /50x.html {
  }
}
}
```
Then I had to make copies of the certificate and key and put them in the directories referenced above in the configuration file. Example:
```
# cp /etc/jupyterhub/server.crt /etc/pki/tls/certs/server.crt
# cp /etc/jupyterhub/server.key /etc/pki/tls/private/server.key
```
Restarted nginx:
```
# systemctl restart nginx
Run the firewall command(s) to allow port 80, 443, and 4443:
# firewall-cmd - add-port=4443/tcp
# firewall-cmd - add-service={http,https}
# firewall-cmd - runtime-to-permanent
```
Change the following line in the jupyterhub_config.py file to port 4443 to match the nginx port:
```
c.JupyterHub.bind_url = 'https://:4443'
```
Or the old deprecated way (still works): 
```
c.JupyterHub.port = 4443
```
At this point I could confirm that HTTPS was working as intended when visiting the proper URL: https://hostname:4443. You should also be redirected here from http://hostname and https://hostname.

There are also other reverse proxies to consider. Such as:
* Haproxy - http://www.haproxy.org (Apparently superior in functionality to nginx)
* Traefik - https://traefik.io/
* Sozu - https://www.sozu.io/

References:
* https://tljh.jupyter.org/en/latest/install/custom-server.html
* https://jupyterhub.readthedocs.io/en/stable/reference/config-sudo.html
* https://github.com/jupyterhub/jupyterhub/wiki/Run-jupyterhub-as-a-system-service
