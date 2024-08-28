---
layout: post
title: "try out jira cli tools"
---

# 背景
公司 bug 管理平台由 bugzilla 切换到 jira 有一阵子了，使用感受: jira 的 dashboard 确实比 bugzilla 好很多，内置的 JQL 也很好用；
但是之前 bugzilla 相关的自动化脚本都不能用了。要想灵活的定制自动化脚本 还是需要 cli 接口，，所以抽时间再看看 jira cli 的实现、用法，，


## Before Start: Generate Token
跟大多数系统一样，在开始 cli 开始之前，需要先创建 Token 用来认证，步骤如下:  
  from webUI: **Click TopRight.Avatar** --> **Profile** --> **Personal Access Tokens**  
创建成功后，在 .bashrc 中增加: **export JIRA_API_TOKEN=$Your_Token_String**  

## 选择安装 jira cli 实现
通过搜索调查，发现目前 **https://github.com/ankitpokhrel/jira-cli** 最火，查找、修改 issue 都很好用；但是还不能满足我们的另一个目标需求：
查看 issue 的 custom field；要满足该需求还需要另一个 jira cli 项目: **https://pypi.org/project/jira-cli** 。  
也就是说：如果要满足我们的所有需求，需要组合使用 Both 两个工具。 他们的安装配置过程分别如下: 

---
### github.com/ankitpokhrel/jira-cli
Install, init and examples:  
```
$ wget https://github.com/ankitpokhrel/jira-cli/releases/download/v1.5.1/jira_1.5.1_linux_x86_64.tar.gz
$ tar axf jira_1.5.1_linux_x86_64.tar.gz
$ sudo cp jira_1.5.1_linux_x86_64/bin/jira /bin/

$ jira init
? Installation type: Local
? Authentication type: bearer
? Link to Jira server: https://issues.redhat.com
? Login username: J*******Yin
? Default project: RHEL
? Default board: All RHEL


$ jira me
$ jira issue list --created  month  --plain  --no-truncate -r$(jira me)
$ jira issue list --created  month  --plain  --no-truncate -q'project = RHEL AND filter = FSQE_Scope AND status != Closed AND "QA Contact" = currentUser()'
$ jira issue list --created  month  --plain  --no-truncate -q'project = RHEL AND filter = FSQE_Scope AND status != Closed AND "QA Contact" = currentUser()' --no-headers --columns KEY,SUMMARY
$ jira issue list --plain --no-truncate --no-headers --columns KEY,SUMMARY  -q"project = RHEL and 'Preliminary Testing' = Requested and 'QA Contact' = currentUser()"
$ jira issue view RHEL-36711 --plain --comments 100
```

---
### pypi.org/project/jira-cli
Install, jirashell usage:  
```
$ pip install jiracli keyring ipython  #keyring,ipython is required by jirashell

$ jirashell -s https://issues.redhat.com
[n] jira = JIRA(server='https://issues.redhat.com', token_auth=os.environ.get('JIRA_API_TOKEN'))
[n] ...
[n] exit()
```

Write python script based on **jira** module:  
```
$ cat /usr/local/bin/jira-issue.py 
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

from __future__ import annotations
import os, sys
from jira import JIRA

jira = JIRA(server='https://issues.redhat.com', token_auth=os.environ.get('JIRA_API_TOKEN'))
allfields=jira.fields()
nameMap = {field['name']:field['id'] for field in allfields}
if len(sys.argv) < 3:
    if sys.argv[1]:
        issue = jira.issue(sys.argv[1])

    print(f"Usage: <{sys.argv[0]}> <issue-id> <field-name [field-name ...]> vvv")
    if issue:
        for key, value in nameMap.items():
            if value in issue.raw['fields']: print(f"  {key:<40}\t{value}")
    else:
        for key, value in nameMap.items(): print(f"  {key:<40}\t{value}")
    print(f"Usage: <{sys.argv[0]}> <issue-id> <field-name [field-name ...]> ^^^\nExamples:")
    print(f"  python {sys.argv[0]} RHEL-24133 'Testable Builds'")
    print(f"  python {sys.argv[0]} RHEL-24133 fixVersions 'Preliminary Testing'")
    exit(1)

issueid = sys.argv[1]
issue = jira.issue(issueid)

def printIssueField(issue, fieldname):
    attrname = fieldname
    if fieldname in nameMap:
        attrname = nameMap[fieldname]
    print(f'=== BEGIN {fieldname}:')
    print(getattr(issue.fields, attrname))
    print(f'--- END {fieldname}:\n')

for field in sys.argv[2:]:
    printIssueField(issue, field)
```


---
### 组合使用
```
#!/bin/bash

command -v jira &>/dev/null || {
	echo "{error} command jira is required!" >&2
	exit 1
}

preTestIssues="$*"
[[ -z "$preTestIssues" ]] && {
	preTestIssues=$(jira issue list --plain --no-truncate --no-headers --columns KEY \
		-q"project = RHEL and 'Preliminary Testing' = Requested and 'QA Contact' = currentUser()")
}

for issue in ${preTestIssues}; do
	echo "${issue}:"
	echo -e $'\E[0;33;44m'"{get-mr-builds-by} jira-issue.py '${issue}' 'Testable Builds'" $'\E[0m'>&2
	issueInfo=$(jira-issue.py "${issue}" 'Testable Builds' fixVersions components)
	echo -e "{debug} issue Testable Builds: ${issueInfo}"|GREP_COLORS='ms=31;47' grep --color Repo.URL:.* >&2

	repos=$(echo "${issueInfo}" | awk '/Repo.URL:.*_debug/{print "   ", $NF}')
	if [[ -n "${repos}" ]]; then
		rhelVersion=$(echo "${issueInfo}" | sed -rn "/BEGIN fixVersions/,/END fixVersions/{/^.*name='([^']+)'.*$/{s//\\1/;p}}")
		component=$(echo "${issueInfo}" | sed -rn "/BEGIN components/,/END components/{/^.*name='([^']+)'.*$/{s//\\1/;p}}")
		echo -e "MR-debug-repos:\n${repos}\nRHEL-version: ${rhelVersion#*-}\nComponent: ${component// /}"
	else
		echo -e "${issue}   #{WARN} can not find MR-build-repo"
	fi
done

```

see also:  
- https://github.com/tcler/bkr-client-improved/blob/master/utils/Jira-issue-need-pre-test.sh
- https://github.com/tcler/bkr-client-improved/blob/master/utils/Jira-issue-need-verify.sh
