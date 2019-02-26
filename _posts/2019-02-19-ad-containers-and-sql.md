### The problem 

Everything appears to be in [containers](https://en.wikipedia.org/wiki/Container_(virtualization)) nowadays, even the SQL Server. But still there are mixed environments, people and companies wanting to try containers without going all in. So I was wondering how practical would it be to have .net core services on docker, running in Windows containers connecting to an external, old fashioned SQL Server instance? Also, as it is all in a Windows domain, I'd like to use domain authentication so I don't have to worry about managing passwords. 

Simple isn't it? Well, it turns not that simple as not everything is on the domain. The SQL Server is, the docker hosts are, but the containers are not.

Additionally there are differences depending on whether you want to run as an independent container, or in docker swarm mode. This blog post focuses on standalone containers, and the swarm mode is covered in <a href="http://dbain.wales/2019/02/22/gmsa-in-swarm-mode/">the follow-up post</a>.

### The quick answer

The good news is that it is not an unreasonable requirement and it has been done before. The solution is to use Group Managed Service Accounts (gMSA) and Credential Spec Files. A number of people have already documented their efforts. Some were more successful than others. 

### My story

My problem was that I wasn't able to make it work just by following any single write-up. In fact, for a few days, I was not able to get it going at all. But eventually it happened and here is a step by step description of how I made it work on Windows Server 2016, 1803, 1809 and 2019 as the host OS and 2016, 1803 and 1809 in full and nano options as the container base image. Generally, it is very simple once you know what to do, and more importantly what not to do (more about it later).

### Test setup

To test it I have set up a virtual lab environment on Azure with 6 VMs

<ul>
<li><strong>DC </strong>- Windows Server 2016 Datacenter acting as a domain controller</li>
<li><strong>DB </strong>- Windows Server 2016 Datacenter with SQL Server 2017 Developer edition installed</li>
<li><strong>DH2016A </strong>- Windows Server 2016 Datacenter with Containers (Docker version 18.09.2)</li>
<li><strong>DH1803A </strong>- Windows Server 1803 with Containers (Docker version 17.06.2-ee-18)</li>
<li><strong>DH1809A</strong> - Windows Server 1809 with Containers (Docker version 18.09.0)</li>
<li><strong>DH2019A</strong> - Windows Server 2019 with Containers (Docker version 18.09.1)</li>
</ul>

I have created a `sqlgmsa.local` domain and joined all the VMs to it. SQL Server was run using `SQLGMSA\SqlServer` Managed Service Account without any special permissions. 

In the domain I have 2 service accounts `SQLGMSA\ServiceA` and `SQLGMSA\ServiceB`. Both have logins on the SQL Server instance. I will be setting some of my containers to connect to the SQL Server as ServiceA and some as ServiceB. 

Initially I tested the connectivity from containers build with full base images (standard mrc.microsoft.com/windows/servercore) and using [dbatools](https://dbatools.io) module to run queries from them to the `DB.sqlgmsa.local server`. To be able to test nano based images I created two test images containing simple .net core WebAPI service written in C# with two public methods. Calling `api/info` you can check if the service is running, what system is it running on. Calling `api/query/` attempts to open connection to the specified database (or master if the db parameter is not provided) and returns information about the database, the original login and the current user. A simple query 

```sql
select 
     @@version SqlServer
    ,db_name()  [Database]
    ,current_user CurrentUser
    ,original_login() OriginalLogin
for json path
```

The [test images](https://hub.docker.com/r/michalporeba/sqlgmsatest/) are available on [docker hub](https://hub.docker.com/) and the [source code here](https://github.com/michalporeba/sharing/tree/master/sql-docker-gmsa/TestService) on [github](https://github.com).

### PowerShell and AD
In the example I am using [PowerShell](https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-6) to manage my active directory. If the commands I use don't work for you you may be missing the AD modules. To install them add the RSAT-AD-PowerShell windows feature by executing this PowerShell command

```powershell
Add-WindowsFeature RSAT-AD-PowerShell
```

### Group Managed Service Accounts (gMSA)
Managed Service Accounts where introduced some time ago to reduce overhead associated with managing passwords for service accounts. The Group Managed Service Accounts solve the same problem but unlike MSAs gMSAs can be used across multiple computers. 

To start using gMSAs on a domain a KDS Root Key has to be created first. It is the key with which passwords shared between the computers on the domain are protected. If your domain has other MSAs already you will not need to do it again. 

To create a KDS Root Key Run I used this PowerShell command on the domain controller
```powershell 
Add-KdsRootKey -EffectiveTime (Get-Date).AddHours(-10)
```
and then to verify that it has been created
```powershell
Get-KdsRootKey
```

Now to create the test service accounts I used the following commands. I was doing it on the DC, but with the right permissions it should be possible to do from any computer on the domain. 

```powershell
New-AdServiceAccount -Name ServiceA -DNSHostName sqlgmsa.local `
   -PrincipalsAllowedToRetrieveManagedPassword "Domain Controllers", "Domain Admins", "CN=DockerHosts,CN=Computers,DC=sqlgmsa,DC=local" `
   -KerberosEncryptionType AES128, AES256

New-AdServiceAccount -Name ServiceB -DNSHostName sqlgmsa.local `
   -PrincipalsAllowedToRetrieveManagedPassword "Domain Controllers", "Domain Admins", "CN=DockerHosts,CN=Computers,DC=sqlgmsa,DC=local" `
   -KerberosEncryptionType AES128, AES256
```

Where ServiceA and ServiceB are the names of the accounts and `"CN=DockerHosts,CN=Computers,DC=sqlgmsa,DC=local"` is the distinguished name of the group I have created for the docker hosts. 

If you don't know the exact distinguished name running this command can help

```powershell 
Get-AdGroup -filter { name -like "yourgroupname" }
```

Now on every docker host all the specific service accounts (2 in my test case) have to be installed so that the host OS can access them. 

```powershell
Install-AdServiceAccount -Identity ServiceA
Install-AdServiceAccount -Identity ServiceB
```

If there is an error message like this, it means the permissions were not set correctly
Install-AdServiceAccount : Cannot install service account. Error Message: '{Access Denied}

### Credential Spec file
[Docker Credential Spec Files](https://success.docker.com/article/modernizing-traditional-dot-net-applications) have been created specifically to solve the problem of passing gMSA to containers. They are plain json files with information about the service account. It is possible to create the files manually, but there is module for it. It is [documented here](https://github.com/MicrosoftDocs/Virtualization-Documentation/tree/live/windows-server-container-tools/ServiceAccounts) but here is a short instruction of how to create get and import the module. 

This part needs to be done on every docker host. 

```powershell 
# 1. Set TLS1.2 support from PowerShell so the module can be downloaded from github. 
PS C:\Tmp> [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

# 2. Download the psm1 file using Invoke-WebRequest
PS C:\Tmp> Invoke-WebRequest "https://raw.githubusercontent.com/MicrosoftDocs/Virtualization-Documentation/live/windows-server-container-tools/ServiceAccounts/CredentialSpec.psm1" -OutFile "CredentialSpec.psm1"

# 3. Import the module
PS C:\Tmp> Import-Module .\CredentialSpec.psm1
```

With CredentialSpec module imported for each gMSA a credential spec file has to be created.

```powershell
PS C:\> New-CredentialSpec -Name ServiceA`
  -AccountName ServiceA `
  -Domain $(Get-AdDomain -Current LocalComputer)

PS C:\> New-CredentialSpec -Name ServiceB`
  -AccountName ServiceA `
  -Domain $(Get-AdDomain -Current LocalComputer)
```

The list of existing files can be obtained with

```powershell
PS C:\> Get-CredentialSpec

Name     Path
----     ----
ServiceA C:\ProgramData\docker\CredentialSpecs\ServiceA.json
ServiceB C:\ProgramData\docker\CredentialSpecs\ServiceB.json
```

And finally, run the containers passing the credential spec files with the `--security-opt` parameter. (This is an example from DH2019A using the 1809 nano base image). 

```powershell 
docker run -d -it -p 8001:80 `
   --security-opt "credentialspec=file://ServiceA.json" `
   --name ServiceA `
   michalporeba/sqlgmsatest:1809nano

docker run -d -it -p 8002:80 `
   --security-opt "credentialspec=file://ServiceB.json" `
   --name ServiceB `
   michalporeba/sqlgmsatest:1809nano
```

### The proof is in the pudding

After checking both containers are running with `docker ps` I can start testing. As the test is not focused on anything else but domain authentication I didn't open any ports to the lab, and all I was doing was to either connect to the container and use dbatools to execute a query on the db server, or from the docker host connecting to the service listening on the published port. Here are the example calls using `Invoke-WebRequest` on DH2019A. 

<img src="https://dbainwales.files.wordpress.com/2019/02/sqlgmsa.proof_.png" alt="sqlgmsa.proof" width="1300" height="390" class="alignnone size-full wp-image-96" />

```powershell
PS C:\> $env:ComputerName
DH2019A
PS C:\> docker start ServiceA
ServiceA
PS C:\> docker start ServiceB
ServiceB
PS C:\> docker ps
CONTAINER ID        IMAGE                               COMMAND                  CREATED             STATUS              PORTS                           NAMES
c6382bb7d816        michalporeba/sqlgmsatest:1809nano   "dotnet TestService.…"   2 days ago          Up 4 seconds        443/tcp, 0.0.0.0:8002->80/tcp   ServiceB
02ece189cb74        michalporeba/sqlgmsatest:1809nano   "dotnet TestService.…"   2 days ago          Up 8 seconds        443/tcp, 0.0.0.0:8001->80/tcp   ServiceA
PS C:\> # Service A
PS C:\> (Invoke-WebRequest -UseBasicParsing http://localhost:8001/api/info).Content
["OS:  Microsoft Windows 10.0.17763 ","Framework: .NET Core 4.6.27317.07"]
PS C:\> (Invoke-WebRequest -UseBasicParsing http://localhost:8001/api/query/DB.sqlgmsa.local).Content
[{"Database":"master","CurrentUser":"guest","OriginalLogin":"SQLGMSA\\ServiceA$"}]
PS C:\> # Service B
PS C:\> (Invoke-WebRequest -UseBasicParsing http://localhost:8002/api/info).Content
["OS:  Microsoft Windows 10.0.17763 ","Framework: .NET Core 4.6.27317.07"]
PS C:\> (Invoke-WebRequest -UseBasicParsing http://localhost:8002/api/query/DB.sqlgmsa.local).Content
[{"Database":"master","CurrentUser":"guest","OriginalLogin":"SQLGMSA\\ServiceB$"}]
PS C:\>
```

### Conclusions

The above setup really boils down to 5 steps. If you want to use windows authentication from windows containers on docker to a SQL Server instance (or a cluster you have to

1. Create gMSAs for your services
2. Create logins for the service accounts on the SQL Server
3. Install gMSAs on the docker hosts
4. Create credential spec files 
5. Create containers with `--security-opt` parameter pointing to the credential spec file. 

That's it. In my case I was able to make it work (for standalone containers, not in swarm mode) on different OS versions (2016, 1803, 1809, 2019) using full and nano base images and using docker 17.06 and 18.09. However, there can be surprises and pulling hair. In the week I spent trying to figure it out I had a number of moments when I thought I've got it, just to realise that what worked a moment ago, doesn't any more. 

The biggest lessons where 

* use AD Groups for managing access to gMSAs rather than individual computer accounts, 
* be very careful with `Set-AdServiceAccount` which I have seen in some of the posts out there, 
* SSPI context errors is not what it seems, and can be very annoying 

More details about [the lessons learnt](https://dbain.wales/2019/02/26/gmsa-and-docker-lessons-learnt/) can be found in [the follow up post](https://dbain.wales/2019/02/26/gmsa-and-docker-lessons-learnt/).

Trying to do the same but in a service run on Docker in swarm mode is similar, but not exactly the same. I have described it <a href="http://dbain.wales/2019/02/22/gmsa-in-swarm-mode/">here</a>.

This post is long enough as it is, so I will not go into the details of those lessons learnt here but instead include them in a follow up to which I will link here later. 

*[AD]: Active Directory
*[gMSA]: Group Managed Service Account
*[KDS]: Key Distribution Service