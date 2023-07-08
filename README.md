<IMG  src="https://www.docker.com/wp-content/uploads/2022/03/horizontal-logo-monochromatic-white.png" style="width: 350px"/>

[[_TOC_]]

#Docker Installation

There's really nothing to it on a Linux Machine :):
```
sudo apt-get install docker.io
```
That's it!

# Docker Configuration

##Using /etc/enviroment file
The /etc/environment is a system-wide configuration file used for setting the environment variables. It is not a shell script, it consists of the assignment expressions, that set the environment variables one per line.

```
sudo nano /etc/environment
```
```
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"
JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
HTTP_PROXY=http://10.131.182.105:8080
HTTPS_PROXY=http://10.131.182.105:8080
```
You can set multiple environment variables in this file. Each environment variable must be in a separate line.
During the system reboot, the environment variables and settings written in this file will automatically be assigned and accessible system-wide.

##Configure Docker Daemon with an add-in settings file for proxy settings, storage location and API Endpoint:

Very important in setting up Docker Daemon consistently is not toching the original configuration files like /lib/systemd/system/docker.service because whenever updates and upgrades are applied through apt update and apt upgrade, this file will get overwritten and the settings are lost whenever there's a new docker.io version, which can lead to a ton of confusion, especially when all your containers are suddenly gone, because the storage location was customized prior to the upgrades. 

The right way to manage any custom settings instead is using a settings add-on fle in the service directory, which the Docker Daemon will automatically pick up during startup.

Stop the docker services:
```
sudo systemctl stop docker.service
sudo systemctl stop docker.socket
```

Create a new directory for the docker.service set up files:
```
sudo mkdir -p /etc/systemd/system/docker.service.d
```
Create a new file for managing these add-on settings:
```
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
```

The following changes are required in this file:
- 2 environment parameters for the HTTP_PROXY and HTTPS_PROXY 
- Add ExecStart -g <root-dir> parameter to determine the new storage location: -g /rtk/docker
In case you haven't assigned a specific filesystem to the /var/lib/docker folder, by default it has only 4GB of space. This is hardly enough to download container images. To change this the following instructions can be followed:

- Add ExecStart -H <IP_Address:PortNumber> (API endpoint for remote access): -H=tcp://0.0.0.0:2375
With this setting tools like portainer can use the docker tcp socket to remotely connect and create, configure and monitor docker containers.
Also deployment scripts like DevOps agents should be able to access the docker instance remotely using this connection.

Add the following settings in the [Service] section:
```yml 
[Service]
Environment="HTTP_PROXY=http://10.131.182.105:8080"
Environment="HTTPS_PROXY=http://10.131.182.105:8080"
ExecStart=
ExecStart=/usr/bin/dockerd -g /rtk/docker -H fd:// -H=tcp://0.0.0.0:2375 --containerd=/run/containerd/containerd.sock
```

Reload the daemon and docker service
```
sudo systemctl daemon-reload
sudo systemctl restart docker
```
Validate that the settings were applied for property 'Environment':
```
sudo systemctl show --property=Environment docker
```
The output shows that the http proxies are picked up:
```
Environment=HTTP_PROXY=http://10.131.182.105:8080 HTTPS_PROXY=http://10.131.182.105:8080
```
Do the same for propery 'ExecStart':
```
sudo systemctl show --property=ExecStart docker
```
The output shows that the root-dir & API endpoint parameters are picked up:
```
ExecStart={ path=/usr/bin/dockerd ; argv[]=/usr/bin/dockerd -g /rtk/docker -H fd:// -H=tcp://0.0.0.0:2375 --containerd=/run/containerd/containerd.sock ; ignore_errors=no ; start_time=[Thu 2023-05-11 08:41:04 UTC] ; stop_time=[n/a] ; pid=155108 ; code=(null) ; status=0/>
```
To validate the above settings were applied you can also run the following command:
```shell
ps aux | grep -i docker | grep -v grep
```

##Testing that docker works
```
sudo docker run hello-world
```

Docker should now download the container and show the output of the hello-world script in the terminal:
```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete
Digest: sha256:ffb13da98453e0f04d33a6eee5bb8e46ee50d08ebe17735fc0779d0349e889e9
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

# Installing Portainer for container management:
## Installation
```
sudo docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```
Wait for the installation to finish:
```
Unable to find image 'portainer/portainer-ce:latest' locally
latest: Pulling from portainer/portainer-ce
772227786281: Pull complete
96fd13befc87: Pull complete
b733663f020c: Pull complete
9fbfa87be55d: Pull complete
Digest: sha256:9fa1ec78b4e29d83593cf9720674b72829c9cdc0db7083a962bc30e64e27f64e
Status: Downloaded newer image for portainer/portainer-ce:latest
945469b7f3d1f433029ed52722e27d9e308a9307103f1583ca0c51b9e570b56e
```
## Configure SSL
### Generate the certificate

To generate certificates on the linux host, openssl can be used directly in the linux prompt, or alternatively you can use the openssl docker container:
```bash
docker run -it -v /c/Code/IT-MFG-Traefik:/data alpine/openssl pkcs12 -in /data/rtk.pfx -clcerts -nokeys -out /data/rtk.crt
```
### Generate the key
```
docker run -it -v /c/Code/IT-MFG-Traefik:/data alpine/openssl rsa -in /data/rtk.pem -out /data/rtk.key -passin pass:<yourpassword>
```
The result is a .crt and .key file

## Upload the certificates to portainer 

Navigate to your hostname at port 9443: https://a724s020.autoexpr.com:9443/

At first log on, you'll be warned that the connection is insecure as we don't have a valid ssl certificate yet. Click "Advanced" and the link "proceed to https://a724s020.autoexpr.com:9443".
![image](https://github.com/stinus/PiHomeLab/assets/16155547/0ce3847c-f157-4d12-a7c1-3d12ff102373)
Next a welcome page will ask you to create an administrator user & password:
![image](https://github.com/stinus/PiHomeLab/assets/16155547/edd1a2bd-4383-4f58-9403-85e6c6c3d245)
Enter the password and store it somewhere safely.

Finally go to the Settings section on the left and upload the crt and key file using the corresponding upload buttons:
![image](https://github.com/stinus/PiHomeLab/assets/16155547/6a184244-53ce-42c8-b8c1-dea876d3933a)

After cleaning your browser cache and restarting chrome, the page should now be accessible without the SSL Certificate warning:
![image](https://github.com/stinus/PiHomeLab/assets/16155547/f969980a-f372-486d-8f40-5fc27144a131)
# Configure NGINX Reversed Proxy with SSL Certificates

Copy the files generated in the step above to a folder on the docker host... for example:
```cmd
\rtk\portainer\certs
```
