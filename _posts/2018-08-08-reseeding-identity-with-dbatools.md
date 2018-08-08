---
layout: post
title:  "Reseeding identity columns with dbatools"
date:   2018-08-08 08:00:00 +0100
categories: dbastuff sqlserver dbatools
permalink: "about/reseeding-identity-columns"
excerpt: "An example of how dbatools proved to be useful once more, this time to quickly reseed all identity columns in many databases in just a few lines of code."
---

Recently I have been asked to increase the current value of all identity columns in tens of databases. No, the point is not to argue if it was smart, or a necessary thing to do, it wasn't production environment and it was helpful to people requesting it, so I focused on how to do it. This blog post is about how [dbatools](https://dbatools.io) once again proved to be an invaluable tool.  

The first instinct was, of course I can do it... just a little bit of dynamic SQL, perhaps a coursor or two and I'll have it done. But the thing is, since I discovered dbatools I don't like dynamic T-SQL any more, and I'm sure whatever the task at hand might be, somebody probably has implemented a dbatools function for that alraedy. In this instance it wasn't not quite so, but there is [Test-DbaIdentityUsage](https://dbatools.io/functions/test-dbaidentityusage/)  which proved to be helpful. With it, in no time, I had code looking like this: 

```powershell
$databases.ForEach{
    $db = $psitem.db
    (Test-DbaIdentityUsage -SqlInstance myinstance -Databases $db).ForEach{
        $newid = [int]($psitem.LastValue * 1.2)
        Write-Host "  processing [$db].[$($psitem.Schema)].[$($psitem.Table)] has max ID $($psitem.LastValue), setting it to $newid"
        Invoke-DbaSqlQuery -SqlInstance myinstance -Database $db -Query "dbcc checkident ('[$($psitem.Schema)].[$($psitem.Table)]', reseed, $newid)"
    }
}
```

Not quite the one-liner I hoped for, but not all bad either. And after few minutes all the identity column next values were increased by 20%. Dbatools made me look smart again ;)