---
title: "Token"
date: 2019-09-09T10:08:17+08:00
draft: false
---

I believe that everyone may be slightly confused when they first encounter the concept of Token. However, I also believe that after you use and understand, you find it very convenient and safe.

As mentioned in [about](/about), there are directories and files in the Bigfile, and each app may not want to give the key to everyone. But it can create a Token, specify a directory, and restrict it to only operate in a directory. When you specify a directory for the token, you don't have to care about the existence of the directory. The directories that do not exist will be created automatically. It should be noted that the directory of the token is based on the root directory.

Moreover, you can set the number of uses for the Token, limit the ip, specify the expiration time, and limit it to be read-only.