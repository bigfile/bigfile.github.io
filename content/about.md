---
title: "About"
date: 2019-09-06T17:07:07+08:00
draft: false
---

[![Build Status](https://travis-ci.org/bigfile/bigfile.svg?branch=master)](https://travis-ci.org/bigfile/bigfile)
[![codecov](https://codecov.io/gh/bigfile/bigfile/branch/master/graph/badge.svg)](https://codecov.io/gh/bigfile/bigfile)
[![GoDoc](https://godoc.org/github.com/bigfile/bigfile?status.svg)](https://github.com/bigfile/bigfile)
[![FOSSA Status](https://app.fossa.io/api/projects/git%2Bgithub.com%2Fbigfile%2Fbigfile.svg?type=shield)](https://app.fossa.io/projects/git%2Bgithub.com%2Fbigfile%2Fbigfile?ref=badge_shield)
[![Go Report Card](https://goreportcard.com/badge/github.com/bigfile/bigfile)](https://goreportcard.com/report/github.com/bigfile/bigfile)
[![Open Source Helpers](https://www.codetriage.com/bigfile/bigfile/badges/users.svg)](https://www.codetriage.com/bigfile/bigfile)

**Bigfile** is a file transfer system, supports http, ftp and rpc protocol. It is built on top of many excellent open source projects. Designed to provide a file management service and give developers more help. At the bottom, bigfile splits the file into small pieces of 1MB, the same slice will only be stored once. Please allow me to illustrate the entire architecture with a picture.

![architecture](/bigfile.png)

In fact, we built a file organization system based on the database. Here you can find familiar files and folders. But in Bigfile, files and folders both are considered to be files.

Since the rpc and http protocols are supported, those languages supported by [grpc](https://grpc.io/) and other languages can be quickly accessed. If you are not a programmer, you can use the ftp client to manage your files, the only thing you need to do is start Bigfile.

It's not `big`， It has `逼格`.  


**Bigfile** is also a system that supports multiple applications, we call it **APP**. Each APP has its own space, and the APPs are isolated from each other without interfering with each other. For security reasons, we don't want to expose the application key to anyone who uses an application, so each **APP** should create a **Token** with a certain permission to manipulate the file. You can restrict the **Token** to only access a directory, set the expiration time, the number of times available, read-only access, and restrict the use of IP. We also provide **HTTPS**, **FTPS** and **RPC** services with double-ended authentication. It is easy to operate and easy to use. The only thing you need to do is to generate a certificate using the command line tools we provide, and specify the certificate when starting the service.