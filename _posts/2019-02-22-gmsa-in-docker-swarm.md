In [my previous post](http://dbain.wales/2019/02/19/ad-containers-and-sql/">) I have explained how I was able to connect from windows containers running on docker to a SQL Server cluster on a network using domain authentication (with gMSAs) rather than SA logins and passwords. 

### gMSAs in docker swarm mode

After I got the containers using Group Managed Service Accounts working on a single Docker host I went on to try the same in the [swarm mode](https://docs.docker.com/engine/swarm/). My plan was to simply replace `docker run -d` part of the command creating the container with `docker service create`, but it turns out that it is not that simple, especially if you don't have a lot of experience with swarm mode. It is also worth noting, that I had a lower success rate than when I was experimenting with standalone containers. I was able to make it work on all the same Windows versions (2016, 1803, 2019, 1809) but only when using Docker 18.09 and not on the 17.06 which was on the image I used for 1803 tests). 

### Demo setup

Similarily to the previous post I have tested this on a range of operating systems and docker versions, but what I want to show here is how it worked on Windows 2019 and Docker 18.09. 

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_1.png" alt="gMSA_Docker_Service_1" width="800" height="338" class="alignnone size-full wp-image-103"/>

To make it a bit more _exciting_ (and because of how Docker Swarm works) this time I will be testing the service from a web browser. To start with it does't work. That is because there is no service listening on port 8101.

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_1_web.png" alt="gMSA_Docker_Service_1_web" width="800" height="410" class="alignnone size-full wp-image-104"/>

### Creating a service

To create a service `docker service create` command is used. When compared to `docker create` some parameters are different, for example there is no `--security-opt` used in the previous post and instead `--credential-spec` is used. But first things first. Let's just create a service using `michalporeba/sqlgmsatest:1809nano` image with minimal configuration and see what happens. 

```powershell 
docker service create -p 8101:80 michalporeba/sqlgmsatest:1809nano
```

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_2.png" alt="gMSA_Docker_Service_2" width="800" height="160" class="alignnone size-full wp-image-105"/>

The `-p 8101:80` makes the service available on the 8101 port using the default _ingress_ network. No errors, the service is running, it is converged, so let's try to connect to it!

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_2_webrequest.png" alt="gMSA_Docker_Service_2_WebRequest" width="800" height="621" class="alignnone size-full wp-image-108"/>

And here is the first surprise. The `localhost` doesn't work. That's a swarm thing and although it is possible to publish ports in _host_ mode it is not how I would be running in production, so I will just open the ports and connect to the service externally using a web browser. 

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_2_web.png" alt="gMSA_Docker_Service_2_web" width="800" height="160" class="alignnone size-full wp-image-106"/>
	
OK, so the `api/info` call was successful. The service from `michalporeba/sqlgmsatest:1809nano image is running and responding. So the next task is to use it to query the `TestDB` database on my test instance `DB.sqlgmsa.local`. 

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_2_web2.png" alt="gMSA_Docker_Service_2_web2" width="800" height="115" class="alignnone size-full wp-image-107"/>

### Adding the gMSA

Not authorized! But who? The `NT AUTHORITY\ANONYMOUS LOGON`. That is because despite the docker host being member of the `sqlgmsa.local` domain, the container running the service is not. To fix it, exactly as in the case of standalone container a Group Managed Service Account has to be created, installed and a credential file created. 

```powershell
# Create gMSA
New-AdServiceAccount -Name MyService -DNSHostName sqlgmsa.local `
  -PrincipalsAllowedToRetrieveManagedPassword "Domain Controllers", "Domain Admins", "CN=DockerHosts,CN=Computers,DC=sqlgmsa,DC=local" `
  -KerberosEncryptionType AES128, AES256

# Install it
Install-AdServiceAccount -Identity MyService

# Import the module to manage Credential Specs
Import-Module .\PsModules\CredentialSpec.psm1

# And create a spec file for MyService
New-CredentialSpec -Name MyService -AccountName MyService `
-Domain (Get-AdDomain -Current LocalComputer)
```

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_3.png" alt="gMSA_Docker_Service_3" width="800" height="386" class="alignnone size-full wp-image-109"/>

Consistency is everything, isn't it? `-Name`, `-Identity`, `-AccountName` on those commands above refer to the same thing, the gMSA name and have to match. The `-Name` parameter on the `New-CredentialSpec` command is used to control the name of the json file containing the credential spec. The filename can be anything, and it doesn't need to match the account name, but I find it easier if it does. The existing credential spec files can be found in `C:\ProgramData\docker\CredentialSpecs\` or by using `Get-CredentialSpec` command from the `CredentialSpec.psm1` module.

The next step is to use the newly created credential spec file when creating the service. The `--security-opt` is not supported when created a service and `--credential-spec` has to be used instead. 

```powershell
docker service create -p 8102:80 `
  --credential-spec file://MyService.json `
  michalporeba/sqlgmsatest:1809nano
```

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_4.png" alt="gMSA_Docker_Service_4" width="800" height="354" class="alignnone size-full wp-image-110"/>

The new service now runs on port `8102` and should use the new `MyService` identity. Let's see. 

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_4_web.png" alt="gMSA_Docker_Service_4_web" width="800" height="100" class="alignnone size-full wp-image-111"/>

Almost there! `Login failed for user SQLGMSA\MyService$` That's good, that means the correct identity has been picked up, so the last thing to do is to create the login on the SQL Server.

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_5_sql.png" alt="gMSA_Docker_Service_5_sql" width="450" height="154" class="alignnone size-full wp-image-112"/>

And now, as if by magic

<img src="https://dbainwales.files.wordpress.com/2019/02/gmsa_docker_service_5_web.png" alt="gMSA_Docker_Service_5_web" width="800" height="329" class="alignnone size-full wp-image-113"/>

The test web service, written in C#, using .net core is hosted in a Docker container running on windows host,and queries a SQL Server database using domain authentication. 