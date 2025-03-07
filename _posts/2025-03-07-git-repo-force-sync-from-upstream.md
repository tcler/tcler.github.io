---
layout: post
title: "git repo: force sync from upstream"
---

# git repo: force sync from upstream

```
#https://stackoverflow.com/questions/9646167/clean-up-a-fork-and-restart-it-from-the-upstream

git remote add upstream /url/to/original/repo
git fetch upstream
git checkout master
git reset --hard upstream/master  
git push origin master --force 
```
