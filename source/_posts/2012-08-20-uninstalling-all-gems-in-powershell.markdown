---
layout: post
title: "Uninstalling All Gems in PowerShell"
date: 2012-08-20 21:43
comments: true
categories: 
---
Simple way
gem list

gem list | %{$_.split(' ')[0]}

describe -aIx

gem list | %{$_.split(' ')[0]} | %{gem uninstall -Iax $_ }

Leaves something to be desired, too many times we have to do %{ xxx yyy $_ }

{% codeblock Straigtforward and painful way lang:ps1 %}
gem list | % { $_ -match "\w+" > $null;$matches[0] } | % { gem uninstall -aIx $_ }
{% endcodeblock %}

{% codeblock Adding xargs to Powershell with a filter lang:ps1  %}
filter xargs { invoke-expression "$args $_" }
{% endcodeblock %}

{% codeblock Better, implementing as a function lang:ps1 %}
function xargs {
  process {
    invoke-expression "$args $_"
  }
}
{% endcodeblock %}

{% codeblock Cleaner, but we still have horrible regex work lang:ps1 %}
gem list | % { $_ -match "\w+" > $null;$matches[0] } | xargs gem uninstall -aIx
{% endcodeblock %}

{% codeblock Add a real filter taking the first lang:ps1 %}
filter Select-FirstWord {
  process {
    $_ | Select-FirstMatching "\S+"
  }
}
filter Select-FirstMatching {
  param($regex)
  process {  
    if($_ -match $regex){$matches[0]}
  }
}
{% endcodeblock %}

{% codeblock Final lang:ps1 %}
gem list | Select-FirstWord | xargs gem uninstall -aIx
{% endcodeblock %}

{% codeblock  lang:ps1 %}
gc dirs | mkdir -path {$_}
{% endcodeblock %}



Explanation: Pipe the lines from “dirs” into invocations of mkdir, on each invocation bind the “path” parameter to the $_ expression. $_ represents piped string value.

So in effect PS has a built-in “xargs” functionality – only much more powerful, but it doesn't work with non-powershell commands
