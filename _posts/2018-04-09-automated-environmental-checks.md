---
layout: post
title:  "Automated Environmental Checks"
date:   2018-04-09 22:42:22 +0100
categories: dbachecks PowerShell Pester
permalink: "about/environmental-checks"
---

### Unit Test Your Environments

[`Infrastructure as Code`](https://en.wikipedia.org/wiki/Infrastructure_as_Code) has been around for quite some time now. There are many clever definitions one can easily find, but for me it is simply an extension of good codinging (or perhaps DevOpsing) principals. You don't go into production systems and keep making changes until it works. You write your code, you check it into your favourite repository, then there is some process that will automatically test it, and eventually it will make its way to production environment. That way if you make a mistake, you can see where you made it and if it happens to work just fine, you can easily do it again, and again, and again as you scale out. 

It's all ~~very~~ easy when you work with 'cloud' virtual environments that wait to ingest next config file and reconfigure themselves automatically. But what if you look after a more traditional setup? Exchange, Share Point, SQL Server clusters hosted on premises on Windows Servers, rather than Docker pods? What if your are not the only admin and people do go in there and make changes to live systems' configuration, perhaps even for good reasons? 

I'd say you do the same as any decent software developer would do when asked to take care of on old, perhaps unfashionable, non-microservice code base: When asked to change anything, start with writing unit tests so you know at the very least you will not make it any worse. That's right, even if you cannot define your configuration with code, you can test it with it. PowerShell and [Pester](https://github.com/pester/Pester) are great tools to do so. And if you happen to be responsible for SQL Servers, then there are unit tests already written for you, available as the [dbachecks](https://dbachecks.io) module. It's MIT licensed. Go, install it and use it. 

### PowerShell and Pester

Pester is an open source test and mock framework for PowerShell. All modern Windows servers have PowerShell, so let's use it. [Jakub Jareš](http://jakubjares.com) one of the contributors to the Pester project has written an [excelent blog post](http://jakubjares.com/2017/12/07/testing-your-environment-tests/) about environmental checks, how to write them in Pester and why you should test you ~~tests~~ checks. That was my starting point when I took it on myself to improve dbachecks. If you haven't yet, go and read it. I will try not to repeat what Jakub wrote there, but rather where I got from there while trying to find a way to structure checks in such a way, that they are both testable, and easily understood by non-developer sysadmins. 

### The Difference 

Unit tests typically follow the AAA pattern. 
* **Arrange** your context
* **Act** (as in perform the action you want to test), and finally 
* **Assert** that the outcome matches your expectations

In unit testing the objective is to test the functionality detached (as much as practical) from the outside environment. Obviously that's something that cannot be done when doing environmental checks. After all, the environments is what we are testing, so the AAA becomes CCC
* **Configure** your context, get the configuration for the the environment you are validating
* **Collect** the data about your environment
* **Confirm** the real life matches your expectations

Describing it like that stopped me going in circles thinking how do I test my tests. Now I'm testing my checks and to do that I follow the AAA pattern. 
* I *arrange* my test by mocking my *configure* and *collect* actions
* I *act* by calling the *confirm* action passing it the mocked details
* I *assert* the outcome by checking if the *confirm* action behaved as expected

Ideally that would be it, write a Pester unit test and then a test for that unit test. In practice it is slightly more complex as we have to work with the limitations of PowerShell and Pester (which after all is primarly for unit test not environmental checks)

### Code Examples

Typically my testable check has 4 components in 3 files. Here is an example from the dbachecks checking Page Verify option.

First we need to write the confirm function and its configuration 
**confirms\Database.PageVerify.ps1** which could be as simple as:
{% highlight PowerShell linenos %}
function Confirm-PageVerify {
    param (
        [parameter(Mandatory=$true,ValueFromPipeline=$true)]
        [object[]]$TestObject
    )
    process {
        $TestObject.PageVerify | Should -Be "CHECKSUM" -Because "we expect Page Verify to be CHECKSUM"
    }
}
{% endhighlight %}

But to be able to test configuration input and to make the Confirm function fit better with the rest of the framework w do
{% highlight PowerShell linenos %}
function Get-ConfigForPageVerifyCheck {
    $pageverifyValidValues = @("NONE", "TORN_PAGE_DETECTION", "CHECKSUM")
    $pageverify = Get-DbcConfigValue policy.pageverify
    if (!($pageverify -in $pageverifyValidValues)) {
        throw "The policy.pageverify is set to $pageverify. Valid values are ($($pageverifyValidValues.Join(", ")))"
    }
    return @{
        PageVerify = (Get-DbcConfigValue policy.pageverify)
    }
}

function Confirm-PageVerify {
    param (
        [parameter(Mandatory=$true,ValueFromPipeline=$true)]
        [object[]]$TestObject, 
        [parameter(Mandatory=$true)][Alias("With")]
        [object]$TestSettings,
        [string]$Because
    )
    process {
        $TestObject.PageVerify | Should -Be $TestSettings.PageVerify -Because $Because
    }
}
{% endhighlight %}

and to make sure it works as expected, we create unit tests for it 
**tests\checks\Database.PageVerify.Tests.ps1** like so
{% highlight PowerShell linenos %}
Describe "Testing Page Verify Confirms" {
    Context "Test configuration" {
        $cases = @(
            @{ Option = "CHECKSUM" },
            @{ Option = "TORN_PAGE_DETECTION" },
            @{ Option = "NONE" }
        )

        It "<Option> is acceptable as policy.pageverify value" -TestCases $cases {
            param($Option) 
            Mock Get-DbcConfigValue { return $Option } -ParameterFilter { $Name -like "policy.pageverify" }
            (Get-ConfigForPageVerifyCheck).PageVerify | Should -Be $Option
        }
        
        It "Throw exception when policy.pageverify is set to unsupported option" {
            Mock Get-DbcConfigValue { return "NOT_SUPPORTED_OPTION" } -ParameterFilter { $Name -like "policy.pageverify" }
            { Get-ConfigForPageVerifyCheck } | Should -Throw 
        }
    }

    Context "Test the confirm function" {
        Mock Get-DbcConfigValue { return "CHECKSUM" } -ParameterFilter { $Name -like "policy.pageverify" }

        $testConfig = Get-ConfigForPageVerifyCheck 

        It "The test should pass when the PageVerify is as configured" {
            @{
                PageVerify = "CHECKSUM"
            } | 
                Confirm-PageVerify -With $testConfig 
        }

        It "The test should fail when the PageVerify is not as configured" {
            {
                @{
                    PageVerify = "NONE"
                } | 
                    Confirm-PageVerify -With $testConfig
            } | Should -Throw 
        }
    }
}
{% endhighlight %}

And finally when we know the configuration and confirm functions are working as expected, the **checks\Database.Tests.ps1** which provides the definition of the check could be as simple as:
{% highlight PowerShell linenos %}
$config = Get-ConfigForPageVerifyCheck                  # Configure
@(Get-Instance).ForEach{                                # Collect on instance level
    @(Get-DatabaseInfo -SqlInstance $psitem).ForEach{   # Collect on database level
        It "$($psitem.Database) should have page verify set to $($config.PageVerify)" {
            # Confirm environmental details (in $psitem)
            $psitem | Confirm-PageVerify -With $config -Because "Page verify helps SQL Server to detect corruption"
        }
    }
}
{% endhighlight %}

But to make it display better, and to provide more information to the curious sysadmin who would like to know what the check does we do this instead:
{% highlight PowerShell linenos %}
Describe "Page Verify Settings Check" {
    $config = Get-SettingsForPageVerifyCheck                    # Configure
    @(Get-Instance).ForEach{                                    # Collect on instance level
        Context "Testing page verify setting on $psitem" {          
            @(Get-DatabaseInfo -SqlInstance $psitem).ForEach{   # Collect on database level
                It "$($psitem.Database) should have page verify set to $($config.PageVerify)" {
                    # Confirm environmental details (in $psitem)
                    $psitem | Confirm-PageVerify -With $config -Because "Page verify helps SQL Server to detect corruption"
                }
            }
        }
    }
}
{% endhighlight %}

If somebody is simply curious what the check checks, and why, all he has to do is to read the strings here or in the Pester output
* "Page Verify Settings Check"
* "Testing page verify setting on <your instance here>"
* "Each <Database> should have page verify set to <Expected Value>"
* "Because page verify helps SQL Server to detect corruption"

