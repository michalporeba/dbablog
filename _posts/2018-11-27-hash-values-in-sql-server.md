## TL;DR
* SQL Server can hash values using some of the common hashing algorithms like MD or SHA.
* It is possible to use XQuery in addition to XPath in XML `value()` function to do things T-SQL cannot do on its own.

## The Details

[Hash values](https://en.wikipedia.org/wiki/Hash_function#Hash_function_algorithms) or (hash codes) is what we typically use to _store_passwords_ in databases. We use salt values too. Definitely, we don't store clear text passwords. Right?

Typically during a login to an application, a password is combined with a salt value stored somewhere in a database and a hash value is calculated which is then compared with the hash value previously stored in a database. But there are other options. It is possible to calculate hash values directly in a database using T-SQL. It could be useful if a bulk update needs to be performed if you want to generate a lot of test users with predefined passwords during a database migration, but it is also possible to overwrite a password hash and gain access to the application as any user.

SQL Server (starting with 2008) has [hashbytes](https://docs.microsoft.com/en-us/sql/t-sql/functions/hashbytes-transact-sql?view=sql-server-2017) function which can be used to calculate hashes using a number of different algorithms. The algorithms supported differ from version to version. The latest 2017 supports `MD2`, `MD4`, `MD5`, `SHA`, `SHA1`, `SHA2_256`, `SHA2_512`.

Let's have a look at how to use it

```SQL
declare @password nvarchar(32) = 'Secret1234'

select 'MD5' Algorithm, hashbytes('md5', @password) Hash
union all select 'SHA' ,hashbytes('sha', @password)
union all select 'SHA2_256' ,hashbytes('sha2_256', @password)
```

which produces something like this

<img class="alignnone size-full wp-image-82" src="https://dbainwales.files.wordpress.com/2018/11/hash1.png" alt="Hashing results" width="830" height="139" />

You can see that the results are of `varbinary` type. That's OK if your application is storing the hash in this format, but from my experiance, most developers will not know that a byte array can be stored in a database and they will convert it into a [Base64](https://en.wikipedia.org/wiki/Base64) string.

SQL Server does not do Base64 encoding (not as far as I know) but it does support XML and XQuery, and they do encoding. So let's use it.

```SQL
declare @password nvarchar(32) = 'Secret1234'

select Algorithm
,convert(xml, N'').value('
xs:base64Binary(xs:hexBinary(sql:column("Hash")))', 'varchar(max)'
) Base64Hash
from (
select 'MD5' Algorithm, hashbytes('md5', @password) Hash
union all select 'SHA' ,hashbytes('sha', @password)
union all select 'SHA2_256' ,hashbytes('sha2_256', @password)
) t
```

Here I'm converting an empty string `N''` to an XML type which creates an empty XML object which allows me to use it's `value()` method to execute [XQuery](https://en.wikipedia.org/wiki/XQuery) which is a functional language. In most examples of XML in SQL Server the only thing you will see is [XPath](https://en.wikipedia.org/wiki/XPath) in `value()` but XQuery can be used too.

The results are more what you'd expect:

<img class="alignnone size-full wp-image-83" src="https://dbainwales.files.wordpress.com/2018/11/hash2.png" alt="Hashes to Base64" width="830" height="138" />

One thing to note. The hashing algorithms operator on bytes so are not only case sensitive but type sensitive too.

```SQL
declare @password nvarchar(32) = 'Secret1234'

select
'varchar' Type
,hashbytes('md5', convert(varchar(32), @password)) Hash
union all select
'nvarchar'
,hashbytes('md5', convert(nvarchar(32), @password))
```

Have a look at the results. Only because type used is different, the hash is different too. 

<img src="https://dbainwales.files.wordpress.com/2018/11/hash3.png" alt="Hash values using different types" width="770" height="105" class="alignnone size-full wp-image-85"/>

If you want to make it work with .Net make sure to use `nvarchar` as in .Net strings are Unicode.

## How can it be useful?

* It is possible to push hash calculation and comparison to database which means the correct hash and salt value are never loaded to the application memory.
* It is possible to generate hashes for test users in database seeding scripts avoiding application doing row by row processing.
* It is possible to generate hashes of objects to detect changes (although the `checksum` function can good enough and faster too).
* It is possible to use XQuery to do things T-SQL cannot do.

But it brings some risk too
* It is possible to gain access to an account by updating values in a database even when using hashing with salt.