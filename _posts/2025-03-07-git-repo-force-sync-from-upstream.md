---
layout: post
title: "git repo: force sync from upstream"
---

# git repo: force sync from upstream

```
#https://stackoverflow.com/questions/9646167/clean-up-a-fork-and-restart-it-from-the-upstream

forceSyncFromUpstream() {
  local uprepo=$1
  local master=${2:-master}
  git remote add upstream $uprepo
  git fetch upstream
  git checkout ${master}
  git reset --hard upstream/${master}  
  git push origin ${master} --force
}

repofrom=
forceSyncFromUpstream $repofrom ${master}

```
