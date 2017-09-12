---
layout: post
title: "bash 中找 substring"
---

### [[]] wildcard
```
[[ "$string" == *"$substring"* ]] && echo match
```

### [[]] regex
```
[[ "$string" =~ $substring ]] && echo match
```

### without [[]]
```
[ -z "${string/*$substring*/}" ] && echo match
```

### stackoverflow
```
https://stackoverflow.com/questions/229551/string-contains-in-bash
```
