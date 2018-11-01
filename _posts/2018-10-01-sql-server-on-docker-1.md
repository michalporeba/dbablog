It is yesterday's news. [SQL Server](https://www.microsoft.com/en-gb/sql-server/sql-server-2017) runs on [Linux](https://www.linux.org/), and on [Docker](https://www.docker.com/) too. There are plenty of blog posts showing how to install, start and connect to it too.

During last Data Relay, I have seen <a href="https://twitter.com/tsqltidy">Mark Pryce-Maher</a>'s talk on SQL Server on Linux (twice). It was a good talk, but it followed the same pattern so not surprisingly a common question from the audience appeared to be 'why would you want to do it?'.

I mean, the biggest claim to fame appears to be that SQL Server actually works on Linux. It doesn't bring any new features. In fact, some features are missing. A good part of docker demos appears to be an exercise in futility too. One starts a container with SQL Server, creates a database, inserts some data then the container restarts and as if by magic the data is gone, the database never existed. So, indeed, why would you want to run an SQL Server on Docker?

This post is my attempt to address this question by showing how I use SQL Server on Docker and how I deal with the persistence problem.

## First: Get it up and running!

If you want to know how to run and connect to SQL Server on Docker read the <a href="https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker?view=sql-server-2017">Microsoft's Quickstart Document</a>, or Andrew Pruski's [blog post](https://dbafromthecold.com/2018/09/25/running-sql-server-2019-ctp-in-a-docker-container/) about running vNext on Docker. 

Making the long story short, assuming you are running Windows, have Docker installed, configured to run Linux containers and that you don't have an SQL Server instance locally installed:

[code lang=PowerShell]
docker pull mcr.microsoft.com/mssql/server:latest
docker run -d -e ACCEPT_EULA=Y -e SA_PASSWORD=Secret1234 `-p 1433:1433 --name sql mcr.microsoft.com/mssql/server:latest
[/code]

That's all you need to do to have the latest production SQL Server (Developer Edition) instance running on localhost on port 1433.

<h2>Creating a database that survives container restarts</h2>

If you have the container from the previous example running let's stop and remove it first.

[code lang=PowerShell]
docker stop sql 
docker rm sql
[/code]

There is a number of ways to persist container data. For my purposes the simples appears to be <a href="https://docs.docker.com/storage/volumes/#start-a-service-with-volumes">docker volumes</a>. You can read more about docker persistence on [the official documentation site](https://docs.docker.com/storage/) but for now let's just create a local volume called <code>sqlvolume</code>

[code lang=PowerShell]
docker volume create sqlvolume
[/code]

and mount that volume on the container

[code lang=PowerShell]
docker run -d -e ACCEPT_EULA=Y -e SA_PASSWORD=Secret1234 -p 1433:1433 --mount source=sqlvolume,target=/data --name sql mcr.microsoft.com/mssql/server:latest
[/code]

Now, if you create a database with files on the mounted volume like so:

[code lang=sql]
create database [TestDb] 
on primary (name = N'TestDb', filename = N'/Data/TestDb.mdf') 
log on (name = N'TestDb_log', filename = N'/Data/TestDb.ldf')
[/code]

and you stop and start the container

[code lang=PowerShell]
docker stop sql 
docker start sql 
[/code]

as if by magic, the <code>TestDb</code> database survived the restart.

<h2>Why I find it useful</h2>

So with some considerable effort, I was able to create a database that survives a restart of a container or that of a host system. Nothing a standard SQL Server instance running on windows couldn't do! So why do I do it?

I write my blog posts on my laptop and I don't want to commit to installing SQL Server on it. I want to be able to try out vNext while still having the ability to run demos against 2017 and I'm definitely not installing two instances on my laptop. Luckily I don't have to. With the above setup all I have to do is run <code>docker start sql</code> and within seconds I have a local instance of SQL Server ready to serve my queries. When I'm done I do <code>docker stop sql</code> and it is as if the SQL Server was never there.