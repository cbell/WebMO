# WebMO Remote Computational Server Setup Guide
---

## Overview 
- Deploying a server hosted in AWS
- Configuring access to server from Webmo main server 
- Adding computational server as a remote host 
- Installing package 
    - GAMESS

> What is not covered: 
> - Setting up main WebMO server 
> - Building a baremetal server 
> - Installing certain interfaces 
> - Enabling suexec on web server*  
> Note: This is not an official source for documentation, but available to the community to help others. 


## Deploying a server hosted in AWS
The easiest way to deploy a server to be hosted in AWS, is to use a CloudFormation Template. There is an example in this repository. Customizations may be needed to the template, but it should cover a basic configuration.    

1. Launch and log into the AWS console.
2. Navigate to the CloudFormation page, and create a new stack
3. Use the researchnode.yaml document to spin up the instance
4. Fill in the parameters to match the desired system. Most likely options to change:
        
    - Volume Size 
    - Instance Size
    - VPC
    - SubnetId
    - Remote SSH Source (need to allow source main server, and any access you require)
    - Hostname
5. Continue the installation, and let it build the CFN stack.


## Configuring access to server from WebMO main server 
> This can easily be done inside of the cloudformation template. The instructions are going to be written out here nonetheless. A second CFN template may be created in the future to expediate any deployments. This document is both for education and to enable teams to deploy the service.*
WebMO uses ssh authentication to setup a remote node, and to run jobs. This will create a user, and then setup the required access. 
### Run on Remote Server 
1. Create user for authentication:

        adduser --home /home/webmo --shell /bin/bash webmo 
### Run on Main server 
2. Log into main server as webmo user
3. Create an ssh-key pair for the user

        ssh-keygen -t rsa 
4. Accept all defaults, and **do not** set a passphrase.
5. Open the public key, and copy to system clipboard

        cat /home/webmo/.ssh/id_rsa.pub
### Run on Remote server 
6. Set maximmum SSH connections (run as root)

        echo 'MaxSessions 10000' > /etc/ssh/sshd_config && systemctl restart sshd
7. Change user to new webmo user

        sudo su webmo
8. Create .ssh folder

        mkdir ~/.ssh
9. Copy the contents of the id_rsa.pub to server

        echo 'paste-information-in-here' >> ~/.ssh/authorized_keys 
10. Change permissions on file 

        chmod 600 ~/.ssh/authorized_keys 
### Run on Main server 
11. Test ssh to remote server - accept remote host ssh-key

        ssh webmo@hostname.fqdn.tld 
    > You may want to manually add the server to the ~/.ssh/known_hosts on the web server. 

## Adding computational server as a remote host 
There are a couple of caveats here. 
- WebMO queue must be stopped. This stops running jobs
- servercontrol.cgi may need to have sleep statements added. Snippet will be added. 

1. Log into WebMO as as the Admin user 
2. Click on System and stop the daemon
3. Click on Remote Server manager 
4. Enter the details that pertain to your system

    - Server Name This will be the display name
    - Hostname This must be publicly and locally resolvable
    - CPU cores  - number of system cores WebMO can run jobs on
    - Username - 'webmo' user that was created on remote server 
    - Home Directory - /home/webmo
    - Scratch directory - this can remain /tmp - or be changed. However make sure that all interfaces use the same scratch directory.
5. Make sure that ssh is the protocol, and click add. This may take a few moments, to a couple minutes. It will create folders and files on the remote system.
6. Log into the remote server with ssh.
7. Change to webmo user, and verify that all files and folders are there

        sudo su webmo
        cd ~
        ls -alFR webmo/
8. A system that was correctly added should have the following directories:

    - ~/webmo
    - ~/webmo/interfaces

There should be various run_$interfacename.cgi files copied into the ~/webmo folder, and $interfacename.int files in the ~/webmo/interfaces folder. 

If there files missing, then you may be seeing a problem that I did during an install. This was resolved by the help of WebMo support. It requires adding multiple sleep(1); commands to the ~/public_html/cgi-bin/webmo/servercontrol.cgi file. The additions will be added in the add_servers function. For IP protection, I'll not add the function to this document. 

I would suggest adding adding a sleep command inside of each loop, and after &copy and &remoteruncommands commands. Example would look like:
        
        forEach
        {
                &copy("$source/$_", "destination/");
                sleep(1);
        }

        or

        &remote_copy_to(\%host, "", "source/*", "destination/");
        sleep(1);
        &remote_copy_to(\%host, "", "source/to/file.example", "destination/to/file.example");
        sleep(1);

You may want to make a copy of the file before you make any changes.

        cp ~/public_html/cgi-bin/webmo/servercontrol.cgi ~/public_html/cgi-bin/webmo/servercontrol.cgi.bak

If you initially added the server, and these files failed to copy - you will need to delete and add the server again. You should not need to restart Apache to get the the changes tracked.  

When the server is being re-added, you should notice the process takes longer. This is tells you that the script changes are in place.  

## Installing interfaces  
### Gamess  

1. First you need to request a copy of the latest installation of Gamess. This can be done using this site: http://www.msg.chem.iastate.edu/GAMESS/download/register/

2. Once access is granted, download the gamess-current.tar.gz to your local machine 

3. SCP the file ti the remote server. 

        scp gamess-current.tar.gz username@remote-server:~

4. Move the file to /usr/local as root

        root@host:~ mv gamess-current.tar.gz /usr/local/

5. Extract files

        cd /usr/local
        tar xzf gamess-current.tar.gz
6. Change permissions on the folder 

        chown -R root:root gamess
        chmod -R g-s gamess
7. Install gfortran and dump version

        sudo apt-get install csh gfortran patch
        gfortran --version

8. Start the config:

    - Follow the on screen prompts
    - Set the machine type to linux64
    - Leave the gamess installation directory as the default
    - Leave the gamess build location as the default
    - Leave the gamess version number as 00
    - Enter "gfortran" for your choice of FORTRAN
    - Enter the two digit (X.X) version of FORTRAN that was the result of the previous command.
    - If you don't have math libraries installed, enter none
    - Use sockets
    - Opt out of LIBEXC
    - No to MSU function
    - Opt out of LIBCCHEM
9. Compile DDI 

        cd ddi
        ./compddi >& compddi.log
        mv ddikick.x ..
        cd ..
10. Compile GAMESS ( This will take about 20 minutes, even on a modern system)

        ./compall >& compall.log

11. Link GAMESS 

        ./lked gamess 00 >& lked.log

12. Edit the /usr/local/gamess/rungms script

        nano /usr/local/gamess/rungms 
13. Edit these fields to match your system:

        set SCR=/tmp
        set USERSCR=/tmp
        set GMSPATH=/usr/local/gamess
>Note that by default they will be to a relative user path, so not changing them at all will cause failures. 

14. Run a test GAMESS job:

        sudo su webmo
        cd ~
        mkdir gamess
        cp -p /usr/local/gamess/tests/standard/exam01.inp ~/gamess
        cd ~/gamess
        /usr/local/gamess/rungms exam01 > exam01.log
> This will change to the webmo user, and then run a test job from the GAMESS installation 

15. Run a test ddikick job:

        cd /usr/local/gamess
        ./ddikick.x ddi/ddi_test.x -ddi 1 1 localhost -scr /tmp
        ./ddikick.x ddi/ddi_test.x -ddi 1 1 hostname.fqdn.tld -scr /tmp
> This will run two tests, first using the localhost hostname. Next will be the resolvable hostname. Both of these need to succeed. Running these can be helpful when trying to determine if the issue is with shared memory or something else. 

16. Enable the engine for the remote server:
    - Login to WebMO as user 'admin'
    - Click 'Interface Manager'
    - Select the remote server from drop down, hit Configure
    - Click the 'Enable' button for GAMESS
    - Click 'Edit' to configure the Gamess interface
    - Verify that the entries are correct; if necessary, edit entries and click Submit
    - Gamess Version: YYYY
    - Gamess directory: /usr/local/gamess
    - Gamess binary (name, not path): gamess.00.x
    - Ddikick binary (name, not path): ddikick.x

**Shared Memory** 

GAMESS needs to have access to large amounts of shared memory on the system. This is something that will almost absolutely need to be ran if you are running medium or larger jobs.  

The error below is a simple indicator of needing to make this modification: 

        DDI Process 0: error code 911
        ddikick.x: application process 0 quit unexpectedly.
        ddikick.x: Fatal error detected.

The default setting should be viewed before any modifications are made:

        cat /proc/sys/kernel/shmmax

This should be recorded somewhere for posterity. 

Increasing to 4GB:
       
        echo 4294967296 > /proc/sys/kernel/shmmax
        echo "kernel.shmmax=4294967296" >> /etc/sysctl.conf
Increasing to 16GB:

        echo 17179869184 > /proc/sys/kernel/shmmax
        echo "kernel.shmmax=17179869184" >> /etc/sysctl.conf

Additional information about this shared memory usage can be found at /usr/local/gamess/ddi/readme.ddi --  Under "Linux on 32 bit processors" -- line 1320 - 1388

There is documentation located here as well:  
- /usr/local/gamess/machines/readme.unix  
- /usr/local/gamess/ddi/readme.ddi  



*May be added in future repository releases
