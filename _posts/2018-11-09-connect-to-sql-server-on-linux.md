How to connect to SQL Server on Linux? 

It really depends on what one means by 'connect'. To connect to the SQL Server instance running on Linux directly or in a container to run a query it is no different from connecting to one running on Windows. Just connect using the DNS name or an IP and a port number if it is not the default 1433. The only difference is that only SQL Authentication is supported so you will not be able to use Windows Authentication and your domain credentials. 

But when managing a SQL Server instance it is sometimes necessary to connect to the operating system on which it runs. This is very different to running SQL Server on Windows. There is no remote desktop, and PowerShell doesn't really work either. That's where the typical Linux admin tools become necessary. 

## SSH to Linux remote server

In a way the [Secure Shell](https://en.wikipedia.org/wiki/Secure_Shell) or SSH is the Linux equivalent of Remote Desktop. There are many ways to do it, especially from another Linux box. Most of us, SQL DBAs, will be typically on a Windows machine. In that case a common approach is to use [PuTTy](https://www.putty.org/). But if you use PowerShell and keep up with updates it is now possible to SSH directly for a PowerShell terminal too. 

```PowerShell
ssh mysqlonlinux.mydomain.com
```
or 
```PowerShell
ssh 10.11.12.13
```

Simple as that. You will connect with a specific user and now most of the admin commands will have to start with `sudo` which allows to execute them with elevated permissions. It's similar to Windows' `Run as Administrator`. 

## Connect to Docker container from the host

If you want to connect to a container running your SQL Server on Linux from the host use `docker exec` to start a bash shell. 

```PowerShell
docker exec -it sql1 bash
```
where `sql1` is the name of the container hosting SQL Server. 

You will connect with superuser permissions and `sudo` will not be necessary. 

## SSH into a container running SQL Server on Linux

If you cannot or don't want to remote onto the host first to connect to the container running your SQL Server instance SSH is the way to go, again. By default, the SQL Server image does not have SSH server installed so a custom image will have to be created. Once that is done it is no different from connecting with the ssh (or PuTTy) to any other Linux server. It doesn't matter that it runs in a container.  