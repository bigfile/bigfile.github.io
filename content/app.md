---
title: "App"
date: 2019-09-09T09:48:01+08:00
draft: false
---

In Bigfile, you can create a lot of APPs, they are logically isolated, have their own space, do not affect each other, each APP can only see the contents of their own directory. We provide a command line tool management app.

In order to create an app, you can run the following command：

    bigfile app:new --name "bigfile blog" --note "for bigfile to upload file"

After success, you will get the App Uid and App secret.

If you want to delete an app, you can run the command: `bigfile app:delete --uid=foobar`, run command `bigfile app:delete` will get more help.

As a system administrator, you may want to view all the apps in the system, it is very simple, just run the command: `bigfile app:list`.


I hope Bigfile will bring you more happiness.
