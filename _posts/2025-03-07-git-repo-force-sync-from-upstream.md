---
layout: post
title: "git repo: force sync from upstream"
---

# git repo: force sync from upstream

```
#https://stackoverflow.com/questions/9646167/clean-up-a-fork-and-restart-it-from-the-upstream

forceSyncFromUpstream() {
  local uprepo=$1
  #local master=$(git remote show origin | sed -n '/HEAD branch/s/.*: //p')
  local master=$(git branch | grep -o -m1 "\b\(master\|main\)\b")

  git remote add upstream $uprepo
  git fetch upstream
  git checkout ${master}
  git reset --hard upstream/${master}  
  git push origin ${master} --force
}

repofrom=
forceSyncFromUpstream $repofrom

```
