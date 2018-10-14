---
layout: post
title:  "Parsing Database Object Names"
date:   2018-08-10 08:00:00 +0100
categories: dbastuff scripting regex powershell
permalink: "about/parsing-database-object-names"
excerpt: "How to parse up to 4 part identifiers with powershell and regex."
---

## Problem: Parse database object identifiers from single name to 4 part name and extract individual elements. So for example `Server1.MyDb.dbo.TableA` is  `TableA` in schema `dbo` in `MyDb` database on a linked server `Server1`.

I'm not looking for a solution that will validate the name is valid, that it meets all the requirements, or even if it is well formatted. I'm assuming that it is a reasonable name, I just want to extract all the components provided: Linked Server, Database, Schema and Object names. 

## Solution 1 - PowerShell and String Splitting

Define a function that splits the string by '.' and assigns values based on number of elements in the array.
```powershell
function Parse-SqlName {
    param([string]$name)
    process {
        $output = [pscustomobject]@{ Server = ""; Database = ""; Schema = ""; Object = ""; }

        $names = $name.Split('.')
        if ($names.Length -ge 4) { $output.Server = $names[-4] }
        if ($names.Length -ge 3) { $output.Database = $names[-3] }
        if ($names.Length -ge 2) { $output.Schema = $names[-2] }
        if ($names.Length -ge 1) { $output.Object = $names[-1] }

        return $output 
    }
}
```

To use it call 
```powershell
Parse-SqlName "Server1.MyDb.dbo.TableA"
```

This returns 
| Server | Database | Schema | Object |
| --- | --- | --- | --- |
| Server1 | MyDb | dbo | TableA |

#Solution 2 - Regular Expression

Regular Expression is another way to parse strings so I set myself a challange to create one that will do the same as the above function and would work regardless how many parts of the identifier are present. Here is the result 

```regex
[\[]?(?:(?:(?:(?<Server>[\w_&@$ -]+)[\.\[\]]+)?(?<Database>[\w_&@$ -]+)[\.\[\]]+)?(?<Schema>[\w_&@$ -]+)[\.\[\]]+)?(?<Object>[\w_&@$ -]+)\]?
```
After processing a string with it up to four groups will be available. It can be used in any language that supports regex, but here is an example using PowerShell again

```powershell
function Parse-SqlName2 {
    param([string]$name)
    process {
        if ($name -match '[\[]?(?:(?:(?:(?<Server>[\w_&@$ -]+)[\.\[\]]+)?(?<Database>[\w_&@$ -]+)[\.\[\]]+)?(?<Schema>[\w_&@$ -]+)[\.\[\]]+)?(?<Object>[\w_&@$ -]+)\]?') {
            return [pscustomobject]@{
                Server = $Matches.Server 
                Database = $Matches.Database 
                Schema = $Matches.Schema 
                Object = $Matches.Object 
            }
        }
    }
}
```