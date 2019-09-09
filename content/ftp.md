---
title: "FTP"
date: 2019-09-09T13:28:42+08:00
draft: false
---

FTP has been around for decades, but even in the 21st century, this ancient technology is still being used because it is very convenient. As long as you start an FTP server, then using the FTP client to manipulate remote server files is like working with local files.

To support this ancient technology, we implemented the ftp protocol to facilitate the transfer of files. And, for security, we support FTPS.

More importantly, the files you upload here can be read using the HTTP protocol and the rpc protocol.


### Start FTP Server

The first step is of course to start the FTP server. You can use the server certificate created in the rpc protocol. If you forget, you can look back and look again.

    bigfile ftp:start --cert-file server.pem --key-file server.key --tls-enable

After successful startup, you will get a message:

    [2019/09/09 13:10:40.758] 21171 DEBUG   Go FTP Server listening on 2121

### Connect to Server

Here you can use the ftp client [FileZilla](https://filezilla-project.org/download.php?type=client), After the installation is successful, set the connection information, and then log in.

![set ftp site info](/ftp-conn-set.png)

After successful conenction, you will see this:

![ftp-connect-succesfully](/ftp-connect-succesfully.png)

You can also login by token:

![ftp-token-connect](/ftp-token-connect.png)

If you meet this, just click `OK`:

![ftp-confirm-cert](/ftp-confirm-cert.png)

After successful login, you will go directly to the root of the token:

![ftp-token-login](/ftp-token-login.png)