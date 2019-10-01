---
title: A Brew Error When Upgrade Mongodb to 4.2
date: 2019-10-01 17:38:05
tags: [Brew, Mongo]
photos: ["../images/mongodb_cover.JPG"]
---

When I tried to upgrade my old mongodb to 4.2 through ` homebrew `:
```
brew tap mongodb/brew
```
<!-- more -->
then:
```
brew install mongodb-community@4.2
```

I encountered a strange error saying:
```
Error: parent directory is world writable but not sticky
```

That is because of the privilege issue of writing to /tmp,
do the following will solve the problem:
```
sudo chmod +t /private/tmp
```

Remeber to link to the new version after the installation:
```
brew link --overwrite mongodb-community
```
