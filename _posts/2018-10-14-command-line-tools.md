_TLDR:_ To efficiently manage multiple SQL Server instances learn [PowerShell](https://github.com/PowerShell/PowerShell/tree/master/docs/learning-powershell) and check out the [dbatools](https://dbatools.io/). There is also the [mssql-cli](https://github.com/dbcli/mssql-cli) to look at too. 

---

Since its introduction in 2005 the SQL Server Management Studio (SSMS) was the tool used for development and administration of SQL Server databases. Most of the common, day to day tasks one can perform by pointing and clicking without the need to remember (or at least type in) all those confusing commands. But I never liked mice much and the main reason I kept using SSMS was that the only real alternative was the `sqlcmd` command line utility which is a command line tool, but doesn't make you any more productive than SSMS. 

But the times are changing. I attended [~SQL~Data Relay](https://www.sqlrelay.co.uk/) earlier this week and it struck me that a lot of _SQL Server_ demos where done without the SSMS. And as I started thinking about it I realised that in fact I don't use it that frequently in my job either. With the power of PowerShell and the `dbatools` and `dbachecks` module I can no longer imagine managing SQL Servers with SSMS. 

## Examples

Imagine you've got 3 servers (just to make the example simple, but it could be 30 or 300, it doesn't really matter). All we have to do is define a variable holding a list of those servers like so: 

```powershell
$s = server1,server2,server3
```

And now we can get to work. Here are a few example of tasks which in SSMS would involve too much clicking for my liking, and task I can perform with a single PowerShell command. 

### Find a Database

I've got hundreds of databases on each instance but I need to find the specific one. All I have to do is 

```powershell 
Find-DbaDatabase -SqlInstance $s -Pattern "PartOfMyDatabaseName"
```

And it really can be just a part of the name, so I don't have to remember the whole name, it doesn't matter if I remember the beginning or the end of the name. 

### Check server logs

Let's say I need to check all my servers error logs for a specific five minutes when our system had a wobble. With SSMS it's hours of mouse abuse. With PowerShell it's a single command again

```powershell
Get-DbaSqlLog -SqlInstance $s -Afater '2018-10-10 10:00' -Before '2018-10-10 10:05`
```

It will collate all error logs for the time period between 10:00 and 10:05 on 10/10/2018. If the output is too long to read in the PowerShell console you can _pipe_ it to `Out-GridView`. In fact you can do that with any powershell command returning data. Try adding `| Out-GridView` at the end of any command like so in the log example 

```powershell
Get-DbaSqlLog -SqlInstance $s -Afater '2018-10-10 10:00' -Before '2018-10-10 10:05` | Out-GridView
```

### Find DB Growth Events

How to find when my databases' files have grown recently? It is possible with SSMS but why would you do it if you can do:

```powershell
Find-DbaDbGrowthEvent -SqlInstance $s | Out-GridView
```

That's it. That is how simple it is. The above command returns all the growth events from all of the servers previously defined in the `$s` variable. 

## Conclusions

I still use SSMS, especially when it comes to managing Availability Groups and the AG Dashboards. It is useful for report writing and some ad hoc querying although for those sort of tasks I started using ~SQL Server Operations Studio~ recently renamed to [Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/what-is?view=sql-server-2017). It is based on VS Code and runs on Windows, Linux and macOS, and although it doesn't have all the features of SQL Server Management Studio, it does have some gems of its own which are not available in SSMS. Another reason could be performance tuning and looking at execution plans, but for those reasons I have long switched to the free Sentry One product [Plan Explorer](https://www.sentryone.com/plan-explorer). 

For anything else I use PowerShell with dbatools. Looking at the content of SQL Relay demos I'm not alone on that trend.  