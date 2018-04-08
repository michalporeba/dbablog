---
layout: post
title:  "TDD with VS Code and Pester"
date:   2018-03-31 16:00:00 +0100
categories: VSCode PowerShell Pester
permalink: "about/vscode-and-pester"
---

I have recently switched to [Visual Studio Code](https://code.visualstudio.com/) for writing [PowerShell](https://docs.microsoft.com/en-us/powershell/) scripts. ([Here is why and how](https://blogs.technet.microsoft.com/heyscriptingguy/2016/12/05/get-started-with-powershell-development-in-visual-studio-code/)). At the same time I started using [Pester](https://github.com/pester/Pester) to do TDD with PowerShell too. It works very well out of the box, at least at the beginning of a project. As you are writing your Pester tests file to cover the function you have just written you can simply hit F5 to execute all of the tests in that file. Easy. 

It's just that it is backwards. The tests should have already been written, and as you are writing your function that will satisfy the tests you try to press F5 and that function gets executed, not the tests. Even more so, if the function is part a module it gets executed out of context. Nothing works. 

Trying to solve my problem I found a very good writeup on [Debugging PowerShell in VS Code](https://blogs.technet.microsoft.com/heyscriptingguy/2017/02/06/debugging-powershell-script-in-visual-studio-code-part-1/) but it did not answer my question of how to do test driven development with pester and Visual Studio Code. 

Eventually to solve my problem I have created [`workinprogress.ps1`](https://gist.github.com/michalporeba/5103a2fae1b1dfa3c1f09b9f4d225420) script which I typically keep just outside of the repository and define the following VS Code configuration in my `launch.json` file.

{% highlight PowerShell %}
{
    "version": "0.2.0",
    "configurations": [

        {
            "type": "PowerShell",
            "request": "launch",
            "name": "Test workinprogress",
            "script": "${workspaceFolder}/../workinprogress.ps1",
            "args": [ "${workspaceFolder}", "${workspaceFolder}/tests", "${file}" ]
        }
    ]
}
{% endhighlight %}

VS Code will pass the current workspace, the path to the unit tests (`${workspaceFolder}/tests` in the example), and the open file with the current line number. Based on that information the following will happen when you hit F5

* if there is a module (.psm1) file in the workspace, the module will be reloaded
* if the open file name ends with `.Tests.ps1` the script will attempt to find the closest pester `-Tag` and execute only that tag from the current file. Failing that the whole pester file will be executed
* if it is not a pester file but the matching (by name) test file exists next to the open file, it will be executed. 
* if there is a matching (by name) test file somewhere inside the tests folder (`${workspaceFoder}/tests` in this example) it will be executed. 

A matching test file has the same name as the original file but ends with `.Tests.ps1` instead of `.ps1`. So for a command file `Get-MyValue.ps1` the matching tests file is `Get-MyValue.Tests.ps1`

The `workinprogress.ps1` file is available on github [here](https://gist.github.com/michalporeba/5103a2fae1b1dfa3c1f09b9f4d225420).