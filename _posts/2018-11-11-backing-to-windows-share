How to back up a database from SQL Server on Linux (perhaps in a Docker container) to a Windows Share already on the network? 

_If you want to know how to run SQL Server on Linux in a Docker container read [my earlier post](https://dbain.wales/2018/11/01/using-sql-server-on-docker-1/)._

### Container run-time privileges and Linux capabilities
Before we start it is important to note that by default the containers are run with very limited privileges and cannot do much. That's by design, to make them more secure. However, that prevents them from being able to use CIFS protocol to mount to an SMB share, exactly what we are trying to do. If you just follow the steps in the next sections when you try to mount the share you will get 

> Unable to apply new capability set.

To avoid this problem the container needs to be created with two extra capabilities: SYS_ADMIN and DAC_READ_SEARCH. You can read more about it on the [here](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities). But to make the long story short you need to add two `--cap-add` parameters to your `docker run` command. 

The full command from my [earlier post](https://dbain.wales/2018/11/01/using-sql-server-on-docker-1/) becomes: 

```PowerShell
docker run -d -e ACCEPT_EULA=Y -e SA_PASSWORD=Secret1234 `
   -p 14333:1433 --name sql1 `
   --cap-add SYS_ADMIN --cap-add DAC_READ_SEARCH ` 
   mcr.microsoft.com/mssql/server:latest
```

Create a container with those capabilities before continuing.

### The Naive Approach
Let's assume you have a network share already available and it is accessible using UNC `\\FileShare1\SqlBackups\`. Being used to Windows networking one would expect to simply take a backup like so, for example: 

```SQL
backup database [TestDb] 
   to disk='\\FileShare1\SqlBackups\TestDB.bak'
```

The command completes with no errors and yet there is no `TestDB.bak` file in `\\FileShare1\SqlBackups`. Stranger still it is possible to restore from that file which doesn't seem to exist. Try: 

```SQL
restore filelistonly from disk='\\FileShare1\SqlBackups\TestDB.bak'
```

It works, because Linux translates the UNC it knows nothing about (after all it's a Windows thing) to a local file `/FileShare/SqlBackups/TestDB.bak`. If you remote to the container (or execute bash on it with `docker exec -it sql bash`) you will find `/FileShare1/SqlBackups/TestDB.bak` file. 

_Interesting, but not what we expected._

### Getting into SMB and CIFS
To solve this problem a network share needs to be _mounted_ to a node in the Linux file system. There are two ways to do it, one is temporary using the `mount` command only, or a more permanent involving editing the `fstab` file and then using `mount`. But before we can do it, one problem has to be solved first. The Windows file server shares the folder using the SMB protocol which is gibberish to Linux. Luckily Linux can be taught to speak CIFS protocol which is compatible with SMB. The way to do it is to install the `cifs-utils` package. On Ubuntu (SQL Server vNext in a container runs on Ubuntu so I use it as an example) is to install the package with `apt-get`. Other flavours of the OS will use their own package managers, but overall the process is the same. 

First, you will have to [Connect to SQL Server on Linux](https://dbain.wales/2018/11/09/connect-to-sql-server-on-linux/). 

Then the CIFS protocol utilities have to be installed using `apt-get` on Ubuntu. 

```bash
apt-get update
apt-get install cifs-utils
```

You will be asked if you want to really do it as it will take an extra 41 MB of disk space. Press `Y` to agree. 

>After this operation, 41.2 MB of additional disk space will be used.  
>Do you want to continue? [Y/n]

Now a directory where the backups will be mounted needs to be created. Typically the external mounts are all in `/mnt`. So let's create a `backups` directory there. 

```bash
mkdir /mnt/backups
```

Now it is time to mount. 

```bash
mount -t cifs //FileShare/SqlBackups /mnt/backups \
   -o username=yourusername,domain=yourdomain,file_mode=0777,dir_mode=0777,rw,sec=ntlm
```

You will be asked for a password and if everything is correct you will get prompt again which means everything went well. It is important to note the change from `\` to `/`. 

It should not be possible to navigate to that directory and create a test text file. 

```bash
cd /mnt/backups
touch test.txt
```

With that, the `test.txt` file should be now visible on `\\FileShare1\Backups`.

And to take the backup to that share becomes no different to taking any other backup to disk:

```SQL
backup database [TestDb] 
   to disk='/mnt/backups/testdb.bak'
```

### Using Shares to Interact with the Host

It is also possible to use this approach instead of using `-v` or `--mount` to share the file system between a container and its host. It is arguably a bit more effort but it allows to easily share backup and restore location between multiple containers without read/write issues associated with folder binding. 