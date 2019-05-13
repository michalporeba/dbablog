There were times when I tried to look for puzzles to solve, especially the T-SQL puzzles (what happened to the T-SQL Challenge site?). Now I don't. Life is challenging as it is, especially if you work with SQL Server and really try to understand what's going on.

So rather than coming up with some contrived problem for you to solve as part of this edition of [T-SQL Tuesday](http://tsqltuesday.com/) (thank you [Matthew McGiffen](https://matthewmcgiffen.com/author/durabledba/)) I will share something that surprised me only last week. And yes, I have solved it already, and will be blogging more about it soon so no there is no big price for solving my production issue here ;)

Here is the scenario

There is a table that stores millions of records. It has a primary key, a date when a record was processed, a bit column indicating whether it was processed or not, and some text fields that are used for something, but in our example, it's just data that takes space on pages.

There is also an application which is using nHibernate to generate a T-SQL query that retrieves one (just one at a time) records from that table where `IsProcessed = 0`. There are 10-50 records like that at peak times, in a table which holds tens of millions of records so making it very, very fast should be easy with a tiny little covering filtered index. Well... it turns out, SQL Server prefers to scan the clustered index instead.

Have a look

The challenge setup

```sql
use tempdb
go
drop table if exists dbo.LongProcessingTable
if not exists(select 1 from sys.tables where name = 'LongProcessingTable')
create table LongProcessingTable (
Id int not null identity primary key
,ProcessedOn datetime2 null
,IsProcessed bit null
,SomeData nvarchar(1024) not null
)

-- just some text to fill up the space on pages
declare @sometext nvarchar(1024) = (
select string_agg(convert(char(1),name), '')
from sys.all_objects
)

-- create just 100k records with some random date values
-- at this time all records are marked as processed
insert into dbo.LongProcessingTable(ProcessedOn, IsProcessed, SomeData)
select top(100000)
dateadd(second, -abs(checksum(a.object_id, b.object_id)%10000), getdate())
,1
,@sometext
from sys.all_objects a
cross join sys.all_objects b

-- now mark 10 rows as not processed
update d set IsProcessed = 0, ProcessedOn = null
from (
select top (10) *
from dbo.LongProcessingTable d
order by ProcessedOn desc
) d
```

Now the query:

```sql
declare @IsProcessed bit = 0

select top(1) Id, SomeData
from dbo.LongProcessingTable
where @IsProcessed = @IsProcessed
```

The above query comes from the application and cannot be changed. It is what it is. And to help you start, here is the index I thought would work, but doesn't.

```sql
create index IX_LongProcessingTable_NotProcessedYet
on dbo.LongProcessingTable(IsProcessed) include (SomeData)
where IsProcessed = 0
```

The index gets ignored and the server goes for the table scan instead.
Of course, there was somebody who discovered it earlier. I wasn't all that surprised that Erik Darling blogged about it in [2015](https://www.brentozar.com/archive/2015/12/filtered-indexes-just-add-includes/), [2017](https://www.brentozar.com/archive/2017/01/filtered-indexes-variables-less-doom-gloom/) and [2018](https://www.brentozar.com/archive/2018/10/filtered-indexes-vs-parameterization-again/) it turns out, he even says 'IT IS KNOWN'... well, it wasn't to me. But even know, with that knowledge, I still cannot change the query, so what can I do? How to make this query more efficient without changing it, and without creating a covering indexing on the whole table which can contain hundreds of GB of data just to get one row.

If you are still reading... well, enjoy the challenge. I will follow up with a few comments and a couple of my attempts at solving the problem later this month (hopefully).