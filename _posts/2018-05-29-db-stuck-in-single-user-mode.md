---
layout: post
title:  "DB Stuck in SINGLE USER Mode"
date:   2018-05-29 20:00:00 +0100
categories: dbastuff sqlserver
permalink: "about/db-stuck-single-user-mode"
excerpt: "What to do when accidently you end up in a situation where one of your databases is stuck in SINGLE_USER mode and some other process is currenlty connected."
---
### Some Background 
_TLDR? Solution is at the bottom of this page._

The internet is full of examples how to close all active connections to a database by doing something like 

```sql
use master
alter database somedb set single_user with rollback immediate
alter database somedb set multi_user 
```

When you try it on your development machine it works just fine, when you do it in QA it is typically OK too, but one day you realise that you have made a mistake, that the `use master` should have been `use somedb` and now you have a production database in a single user mode, there is only one connection allowed at the time and yours is not the one. Worse, there are hundreds of clients, all queueing to be the only user!

You cannot connect to the database 
```
Msg 924, Level 14, State 1, Line 1
Database 'somedb' is already open and can only have one user at a time. 
```

Trying to do anything that starts with `alter database ...` will fail too
```
Msg 5064, Level 16, State 1, Line 1
Changes to the state or options of database 'somedb' cannot be made at this time. The database is in single-user mode, and a user is currently connected to it. 
Msg 5069, Level 16, State 1, Line 1
ALTER DATABASE statement failed. 
```

Trying to get some ‘authority’ over the server by connecting as the admin (DAC) is of no use either. 

So what can be done? If you happen to have one sql login per database (or a handful) then you can revoke that logins permission to log in, but then you have to make sure you grant it back, to all of the ones you revoked it from. It would work but typically either there are too many logins that would have to have rights revoked or there are too few logins and revoking any rights would make the outage wider than it already is. 

A fairly obvious solution is to shut down the server and bring it back up in the single user mode. The `Sqlservr.exe -m ` option. Then only one admin user will be allowed to connect to the instance, which will allow you to change the permissions of the affected database. The only problem is, that we were talking about a problem which affects almost exclusively busy production server. Shutting it down to fix just one database sounds like an overkill and I bet there will be a lot of explaining to do afterwards.  

### Solution

What I found to work quite well is to open two sessions. In the first one I execute a query that will constantly attempt to connect to the database which is stuck in the single suer mode 

```sql 
use somedb
go 200
```

That ensures that I've got a session that constantly is trying to connect. Then in the other session I will look up the SPID of the session currently connected and kill it. Obviously that assumes that I know my system, and I know that the application will recover if I kill the session. 

```sql 
select spid from sys.sysprocesses where dbid = db_id('somedb')
kill <spid>
```

Do it a couple of times making sure you don't kill the spid of the first session and your first session should be connected to the database. Now all that is left is to finally do 

```sql
alter database somedb set multi_user
```