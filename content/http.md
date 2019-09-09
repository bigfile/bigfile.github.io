---
title: "HTTP"
date: 2019-09-08T21:30:11+08:00
draft: false
---

The http protocol is the first protocol we implemented. In this way, most languages can access Bigfile at almost no cost. In http protocol, we have the following characteristics:

1. record every access rquest (default: enable)
2. parameter signature verification (default: enable)
3. rate limit by ip (default: disable)
4. prevent replay attacks (default: enable)

Next, we will make an example of the use of each API.


### Start Http Server

You can start the http service with the Bigfile command line tool, like this:

    bigfile http:start --cert-file bigfile.pem --cert-key bigfile.key

After the startup is successful, you will see a message. 

	[2019/09/09 09:10:41.078] 14412 INFO  bigfile http service listening on: https://0.0.0.0:10985

You can view the current http supported routes in the following way.

    bigfile http:routes   

This command will render a table that contains the method, path and handler.

    +--------+-----------------------------+------------------------------------------------------+
    | METHOD |            PATH             |                       HANDLER                        |
    +--------+-----------------------------+------------------------------------------------------+
    | POST   | /api/bigfile/token/create   | github.com/bigfile/bigfile/http.TokenCreateHandler   |
    | POST   | /api/bigfile/file/create    | github.com/bigfile/bigfile/http.FileCreateHandler    |
    | PATCH  | /api/bigfile/token/update   | github.com/bigfile/bigfile/http.TokenUpdateHandler   |
    | PATCH  | /api/bigfile/file/update    | github.com/bigfile/bigfile/http.FileUpdateHandler    |
    | DELETE | /api/bigfile/token/delete   | github.com/bigfile/bigfile/http.TokenDeleteHandler   |
    | DELETE | /api/bigfile/file/delete    | github.com/bigfile/bigfile/http.FileDeleteHandler    |
    | GET    | /api/bigfile/file/read      | github.com/bigfile/bigfile/http.FileReadHandler      |
    | GET    | /api/bigfile/directory/list | github.com/bigfile/bigfile/http.DirectoryListHandler |
    +--------+-----------------------------+------------------------------------------------------+

### Parameter Signature

Before you start using it, what you need to know is how to sign the parameters. The rules are actually very simple. All parameters, except binary data, are sorted alphabetically, forming key-value pairs in the form of `Key=Value`, then using `&` matching, finally concating the password, and then performing an MD5 calculation, which is over. 

for example,you password is `46afc3607a93ac410357a8ed53a872b8`， and you have these parameters, `name=bigfile`, `age=25`, and `gender=body`. 

After sort and concat is: `age=25&gender=body&name=bigfile46afc3607a93ac410357a8ed53a872b8`

Running the md5 algorithm, pseudo code is: `md5("age=25&gender=body&name=bigfile46afc3607a93ac410357a8ed53a872b8")`. Assume the result is `f8f2ae1fe4f70b788254dcc991a35558`.

Then you should submit these parameters: `age=25&gender=body&name=bigfile&sign=f8f2ae1fe4f70b788254dcc991a35558`.

Very simple, isn't it?



### Token Create

Creating a Token is the beginning of all subsequent operations. Before we start, let's take a look at the parameters of this API.

|name|type|required|description|
|:---:|:--:|:--:|:--:|
|appUid|string|yes|app uid|
|nonce|string|yes|a random string, length: `32-48`|
|sign|string|yes|the signature of parameters|
|path|string|no|default: `/`, the scope of token|
|ip|string|no|limit the ip that uses this token, multiple ips separated by commas|
|expiredAt|timestamp|no|default permanent|
|secret|string|no|token secret, length: `32`|
|availableTimes|int|no|default permanent|
|readOnly|bool|no|whether this token is only be used to download file|

For security, we recommend setting the expiration time and scope for each token. Let's look at a complete example:

```go
package main

import (
	"fmt"
	"io/ioutil"
	libHttp "net/http"
	"strings"
	"time"

	"github.com/bigfile/bigfile/databases/models"
	"github.com/bigfile/bigfile/http"
)

func main() {
	appUid := "42c4fcc1a620c9e97188f50b6f2ab199"
	appSecret := "f8f2ae1fe4f70b788254dcc991a35558"
	body := http.GetParamsSignBody(map[string]interface{}{
		"appUid":         appUid,
		"nonce":          models.RandomWithMd5(128),
		"path":           "/images/png",
		"expiredAt":      time.Now().AddDate(0, 0, 2).Unix(),
		"secret":         models.RandomWithMd5(44),
		"availableTimes": -1,
		"readOnly":       false,
	}, appSecret)
	request, err := libHttp.NewRequest(
		"POST", "https://127.0.0.1:10985/api/bigfile/token/create", strings.NewReader(body))
	if err != nil {
		fmt.Println(err)
		return
	}
	request.Header.Set("Content-Type", "application/x-www-form-urlencoded")
	resp, err := libHttp.DefaultClient.Do(request)
	if err != nil {
		fmt.Println(err)
		return
	}
	if bodyBytes, err := ioutil.ReadAll(resp.Body); err != nil {
		fmt.Println(err)
		return
	} else {
		fmt.Println(string(bodyBytes))
	}
}
```

After request, you will get a response like this:

```json
{
    "requestId":10001,
    "success":true,
    "errors":null,
    "data":{
        "availableTimes":-1,
        "expiredAt":1568127304,
        "ip":null,
        "path":"/images/png",
        "readOnly":0,
        "secret":"46afc3607a93ac410357a8ed53a872b8",
        "token":"4d50ae8061c1d6f148a45031356294bd"
    }
}
```

### Token Update

When a token is created, we can update it as needed. Let's take a look at the parameters of this API. In addition to which token needs to be pointed out, the other parameters are almost indentical to the creation of the token.

|name|type|required|description|
|:---:|:--:|:--:|:--:|
|appUid|string|yes|app uid|
|token|string|yes|the token needs to be updated|
|nonce|string|yes|a random string, length: `32-48`|
|sign|string|yes|the signature of parameters|
|path|string|no|default: `/`, the scope of token|
|ip|string|no|limit the ip that uses this token, multiple ips separated by commas|
|expiredAt|timestamp|no|default permanent|
|secret|string|no|token secret, length: `32`|
|availableTimes|int|no|default permanent|
|readOnly|bool|no|whether this token is only be used to download file|

There's a complete exmaple:

```go
package main

import (
	"fmt"
	"io/ioutil"
	libHttp "net/http"
	"strings"
	"time"

	"github.com/bigfile/bigfile/databases/models"
	"github.com/bigfile/bigfile/http"
)

func main() {
	appUid := "42c4fcc1a620c9e97188f50b6f2ab199"
	appSecret := "f8f2ae1fe4f70b788254dcc991a35558"
	body := http.GetParamsSignBody(map[string]interface{}{
		"appUid":         appUid,
		"token":          "4d50ae8061c1d6f148a45031356294bd",
		"nonce":          models.RandomWithMd5(128),
		"path":           "/images/png",
		"expiredAt":      time.Now().AddDate(0, 0, 10).Unix(),
		"secret":         models.RandomWithMd5(44),
		"availableTimes": -1,
		"readOnly":       false,
	}, appSecret)
	request, err := libHttp.NewRequest(
		libHttp.MethodPatch, "https://127.0.0.1:10985/api/bigfile/token/update", strings.NewReader(body))
	if err != nil {
		fmt.Println(err)
		return
	}
	request.Header.Set("Content-Type", "application/x-www-form-urlencoded")
	resp, err := libHttp.DefaultClient.Do(request)
	if err != nil {
		fmt.Println(err)
		return
	}
	if bodyBytes, err := ioutil.ReadAll(resp.Body); err != nil {
		fmt.Println(err)
		return
	} else {
		fmt.Println(string(bodyBytes))
	}
}
```

After request, you will get a response like this:

```json
{
    "requestId":10003,
    "success":true,
    "errors":null,
    "data":{
        "availableTimes":-1,
        "expiredAt":1568819447,
        "ip":null,
        "path":"/images/png",
        "readOnly":0,
        "secret":"471c983b1e7052ef6a3ed4bd8b3bb42b",
        "token":"4d50ae8061c1d6f148a45031356294bd"
    }
}
```

In this example, we update the expired time. **Note that the Http method is `PATCH` here.**


### Token Delete 

When a token is no longer used, we should delete it. The pamameters for deleting a token is very simple.

|name|type|required|description|
|:---:|:--:|:--:|:--:|
|appUid|string|yes|app uid|
|token|string|yes|the token needs to be updated|
|nonce|string|yes|a random string, length: `32-48`|
|sign|string|yes|the signature of parameters|

But you need to pay attention to is that HTTP method here is `DELETE`, and according to the HTTP protocol, the body is not parsed in the delete method, so we should put the parameters in the query string. A complete example is as follow:

```go
package main

import (
	"fmt"
	"io/ioutil"
	libHttp "net/http"
	"strings"

	"github.com/bigfile/bigfile/databases/models"
	"github.com/bigfile/bigfile/http"
)

func main() {
	appUid := "42c4fcc1a620c9e97188f50b6f2ab199"
	appSecret := "f8f2ae1fe4f70b788254dcc991a35558"
	qs := http.GetParamsSignBody(map[string]interface{}{
		"appUid": appUid,
		"token":  "4d50ae8061c1d6f148a45031356294bd",
		"nonce":  models.RandomWithMd5(128),
	}, appSecret)
	url := "https://127.0.0.1:10985/api/bigfile/token/delete?" + qs
	request, err := libHttp.NewRequest(libHttp.MethodDelete, url, strings.NewReader(""))
	if err != nil {
		fmt.Println(err)
		return
	}
	resp, err := libHttp.DefaultClient.Do(request)
	if err != nil {
		fmt.Println(err)
		return
	}
	if bodyBytes, err := ioutil.ReadAll(resp.Body); err != nil {
		fmt.Println(err)
		return
	} else {
		fmt.Println(string(bodyBytes))
	}
}
```

After request, you will get a response like this:

```json
{
    "requestId":10004,
    "success":true,
    "errors":null,
    "data":{
        "availableTimes":-1,
        "deletedAt":1567956116,
        "expiredAt":1568819447,
        "ip":null,
        "path":"/images/png",
        "readOnly":0,
        "secret":"471c983b1e7052ef6a3ed4bd8b3bb42b",
        "token":"4d50ae8061c1d6f148a45031356294bd"
    }
}
```

The field `deletedAt` is existed and not null, indicating that the token has been deleted.


In the next examples, if your token doesn't have a password, you don't have to sign it.

### File Create

In Bigfile, file and directory both are considered to be `File`, so with this API, you can upload files or create dirctories.


|name|type|required|description|
|:---:|:--:|:--:|:--:|
|token|string|yes|token value|
|nonce|string|yes|a random string, length: `32-48`|
|path|string|yes|the file storage path, it's relative to the path of token |
|sign|string|no|the signature of parameters. it's required if the token has password|
|hash|string|no|the hash of file, based on sha256 algorithm|
|size|int|no|the size of file|
|overwrite|bool|no|indicates that the file is overwritten if it exists|
|rename|bool|no|indicates that the file is renamed if it exists|
|append|bool|no|indicates that the file is appended if it exists|
|hidden|bool|no|Indicates whether the file is hidden or not, and the hidden file cannot be downloaded.|


```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"io/ioutil"
	"mime/multipart"
	libHttp "net/http"
	"os"

	"github.com/bigfile/bigfile/databases/models"
	"github.com/bigfile/bigfile/http"
)

func main() {
	token := "49f92acd696260abba1bc4062d157199"
	tokenSecret := "9bcac735fd7f25947a3909998420affa"

	file, err := os.Open("/Users/fudenglong/Downloads/WechatIMG455.jpeg")
	if err != nil {
		fmt.Println(err)
		return
	}
	for index := 0; ; index++ {
		var (
			err            error
			body           = new(bytes.Buffer)
			chunk          = make([]byte, models.ChunkSize)
			request        *libHttp.Request
			readCount      int
			formBodyWriter = multipart.NewWriter(body)
			formFileWriter io.Writer
		)
		if readCount, err = file.Read(chunk); err != nil {
			fmt.Println(err)
			return
		}
		params := map[string]interface{}{
			"token": token,
			"path":  "/profile/WechatIMG455.jpeg",
			"nonce": models.RandomWithMd5(255),
		}
		if index == 0 {
			params["overwrite"] = "1"
		} else {
			params["append"] = "1"
		}
		params["sign"] = http.GetParamsSignature(params, tokenSecret)
		for k, v := range params {
			if err = formBodyWriter.WriteField(k, v.(string)); err != nil {
				fmt.Println(err)
				return
			}
		}
		if formFileWriter, err = formBodyWriter.CreateFormFile("file", "random.bytes"); err != nil {
			fmt.Println(err)
			return
		}
		if _, err = formFileWriter.Write(chunk[:readCount]); err != nil {
			fmt.Println(err)
			return
		}
		if err = formBodyWriter.Close(); err != nil {
			fmt.Println(err)
			return
		}

		api := "https://127.0.0.1:10985/api/bigfile/file/create"
		if request, err = libHttp.NewRequest(libHttp.MethodPost, api, body); err != nil {
			fmt.Println(err)
			return
		}
		request.Header.Set("Content-Type", formBodyWriter.FormDataContentType())
		resp, err := libHttp.DefaultClient.Do(request)
		if err != nil {
			fmt.Println(err)
			return
		}

		if bodyBytes, err := ioutil.ReadAll(resp.Body); err != nil {
			fmt.Println(err)
			return
		} else {
			fmt.Println(string(bodyBytes))
		}

	}
}
```

In this example, we uploaded a 4.7MB image. Since Bigfile only allows a maximum of 1MB of files to be uploded, we split the file into 1MB pieces and upload it. Moreover, when uploading the first chunk, if the specified path already exists, we overwrite it. Subsequent chunks can be appended to the path.

In this upload, we will get 5 responses，let's see the last response：

```json
{
    "requestId":10020,
    "success":true,
    "errors":null,
    "data":{
        "ext":"jpeg",
        "fileUid":"64c8a8ecd911630acf1dc26e8319f2dd",
        "hash":"9536467bde347627e27634a77963105a045f624e290b0f2bbc342834abdd4593",
        "hidden":0,
        "isDir":0,
        "path":"/images/png/profile/WechatIMG455.jpeg",
        "size":4698744
    }
}
```

From the response we can see that the file is stored in path `/images/png/profile/WechatIMG455.jpeg`. And the hash of the whole file is `9536467bde347627e27634a77963105a045f624e290b0f2bbc342834abdd4593`. This can be used by us to verify the integrity of the file. `isDir=0` indicates we create a file, not directory. When you don't set the `file` field, you will create a directory. Let's see a example:

```go
package main

import (
	"bytes"
	"fmt"
	"io/ioutil"
	"mime/multipart"
	libHttp "net/http"

	"github.com/bigfile/bigfile/databases/models"
	"github.com/bigfile/bigfile/http"
)

func main() {
	var (
		err            error
		body           = new(bytes.Buffer)
		request        *libHttp.Request
		token          = "49f92acd696260abba1bc4062d157199"
		tokenSecret    = "9bcac735fd7f25947a3909998420affa"
		formBodyWriter = multipart.NewWriter(body)
	)
	params := map[string]interface{}{
		"token": token,
		"path":  "/profiles",
		"nonce": models.RandomWithMd5(255),
	}
	params["sign"] = http.GetParamsSignature(params, tokenSecret)
	for k, v := range params {
		if err = formBodyWriter.WriteField(k, v.(string)); err != nil {
			fmt.Println(err)
			return
		}
	}
	if err = formBodyWriter.Close(); err != nil {
		fmt.Println(err)
		return
	}

	api := "https://127.0.0.1:10985/api/bigfile/file/create"
	if request, err = libHttp.NewRequest(libHttp.MethodPost, api, body); err != nil {
		fmt.Println(err)
		return
	}
	request.Header.Set("Content-Type", formBodyWriter.FormDataContentType())
	resp, err := libHttp.DefaultClient.Do(request)
	if err != nil {
		fmt.Println(err)
		return
	}

	if bodyBytes, err := ioutil.ReadAll(resp.Body); err != nil {
		fmt.Println(err)
		return
	} else {
		fmt.Println(string(bodyBytes))
	}
}
```

This will get a different response than uploading the file.

```json
{
    "requestId":10025,
    "success":true,
    "errors":null,
    "data":{
        "fileUid":"03ccc6c26ec7c3a0fe2be90c3f3882d0",
        "hidden":0,
        "isDir":1,
        "path":"/images/png/profiles",
        "size":0
    }
}
```


### File Update

After the file is uploaded, we can update a file, mainly used to move the file to another path, let's try.

|name|type|required|description|
|:---:|:--:|:--:|:--:|
|token|string|yes|the token that can access to this file|
|fileUid|string|yes|file uid|
|nonce|string|yes|a random string, length: `32-48`|
|path|string|no|the file storage path, it's relative to the path of token |
|sign|string|no|the signature of parameters. it's required if the token has password|
|hidden|bool|no|Indicates whether the file is hidden or not, and the hidden file cannot be downloaded.|


```go
package main

import (
	"fmt"
	"io/ioutil"
	libHttp "net/http"
	"strings"

	"github.com/bigfile/bigfile/databases/models"
	"github.com/bigfile/bigfile/http"
)

func main() {
	token := "49f92acd696260abba1bc4062d157199"
	tokenSecret := "9bcac735fd7f25947a3909998420affa"
	body := http.GetParamsSignBody(map[string]interface{}{
		"token":   token,
		"fileUid": "64c8a8ecd911630acf1dc26e8319f2dd",
		"nonce":   models.RandomWithMd5(128),
		"path":    "/profile/profile.jpeg",
	}, tokenSecret)
	request, err := libHttp.NewRequest(
		libHttp.MethodPatch, "https://127.0.0.1:10985/api/bigfile/file/update", strings.NewReader(body))
	if err != nil {
		fmt.Println(err)
		return
	}
	request.Header.Set("Content-Type", "application/x-www-form-urlencoded")
	resp, err := libHttp.DefaultClient.Do(request)
	if err != nil {
		fmt.Println(err)
		return
	}
	if bodyBytes, err := ioutil.ReadAll(resp.Body); err != nil {
		fmt.Println(err)
		return
	} else {
		fmt.Println(string(bodyBytes))
	}
}
```

We will get a response like this:

```json
{
    "requestId":10021,
    "success":true,
    "errors":null,
    "data":{
        "ext":"jpeg",
        "fileUid":"64c8a8ecd911630acf1dc26e8319f2dd",
        "hash":"9536467bde347627e27634a77963105a045f624e290b0f2bbc342834abdd4593",
        "hidden":0,
        "isDir":0,
        "path":"/images/png/profile/profile.jpeg",
        "size":4698744
    }
}
```

As you can see from the above results, the file is moved to another path, but the file uid never changes.


### File Read

*File Read* is used to download a file, the file is automatically downloaded when you open the download URL in your browser. If you want to preview the file in your browser, you can set `openInBrowser` to `true`.

*File Read* also supports [HTTP range requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests), but only for *Single part ranges*.

|name|type|required|description|
|:---:|:--:|:--:|:--:|
|token|string|yes|the token that can access to this file|
|fileUid|string|yes|file uid|
|nonce|string|no|a random string, length: `32-48`|
|sign|string|no|the signature of parameters. it's required if the token has password|
|openInBrowser|bool|no|preview the file in your browser|

Since my token has a password, I need to sign it to generate a download link.

>   https://127.0.0.1:10985/api/bigfile/file/read?fileUid=64c8a8ecd911630acf1dc26e8319f2dd&token=49f92acd696260abba1bc4062d157199&sign=f1f67cb50ff890e9b537f0afcaaa8ec4

### Directory List

As the name suggests, is used to view the contents of the directory. Which will list all subdirectories and files in the directory. 

|name|type|required|description|
|:---:|:--:|:--:|:--:|
|token|string|yes|the token that can access to this file|
|nonce|string|yes|a random string, length: `32-48`|
|sign|string|no|the signature of parameters. it's required if the token has password|
|subDir|string|no|default: `/`, list the path of the token|
|sort|string|no|default: `-type`, eg: `-type, name, -name, time and -time`|
|limit|int|no|default: `10`, min: `10`, max: `20`|
|offset|int|no|default: `0`|

The directory may be made up of a lot of content, so we need to get them by pagination.

```go
package main

import (
	"fmt"
	"io/ioutil"
	libHttp "net/http"
	"strings"

	"github.com/bigfile/bigfile/databases/models"
	"github.com/bigfile/bigfile/http"
)

func main() {
	var (
		err         error
		request     *libHttp.Request
		token       = "49f92acd696260abba1bc4062d157199"
		tokenSecret = "9bcac735fd7f25947a3909998420affa"
	)
	params := map[string]interface{}{
		"token": token,
		"nonce": models.RandomWithMd5(255),
	}
	body := http.GetParamsSignBody(params, tokenSecret)

	api := "https://127.0.0.1:10985/api/bigfile/directory/list?" + body
	if request, err = libHttp.NewRequest(libHttp.MethodGet, api, strings.NewReader("")); err != nil {
		fmt.Println(err)
		return
	}
	resp, err := libHttp.DefaultClient.Do(request)
	if err != nil {
		fmt.Println(err)
		return
	}

	if bodyBytes, err := ioutil.ReadAll(resp.Body); err != nil {
		fmt.Println(err)
		return
	} else {
		fmt.Println(string(bodyBytes))
	}
}
```

We will get such a response:

```json
{
    "requestId":10028,
    "success":true,
    "errors":null,
    "data":{
        "items":[
            {
                "fileUid":"034ef654f7e787240cac901c197f09cc",
                "hidden":0,
                "isDir":1,
                "path":"/images/png/profile",
                "size":4698744
            },
            {
                "fileUid":"03ccc6c26ec7c3a0fe2be90c3f3882d0",
                "hidden":0,
                "isDir":1,
                "path":"/images/png/profiles",
                "size":0
            }
        ],
        "pages":1,
        "total":2
    }
}
```

But did not see the file we just created, because it's not in the `/images/png` directory. If you want to list it, you need to set the param `subDir=/profile`. If you try, you will get the following results:

```json
{
    "requestId":10029,
    "success":true,
    "errors":null,
    "data":{
        "items":[
            {
                "ext":"jpeg",
                "fileUid":"64c8a8ecd911630acf1dc26e8319f2dd",
                "hash":"9536467bde347627e27634a77963105a045f624e290b0f2bbc342834abdd4593",
                "hidden":0,
                "isDir":0,
                "path":"/images/png/profile/profile.jpeg",
                "size":4698744
            }
        ],
        "pages":1,
        "total":1
    }
}
```

### File Delete

*File Delete* will delete a directory or file. When you delete the directory, you need to be very careful, you may delete all the files in the directory. Deleted files and directories are actually recoverable, but at the moment, we have not developed the trash can function.

|name|type|required|description|
|:---:|:--:|:--:|:--:|
|token|string|yes|the token that can access to this file|
|nonce|string|yes|a random string, length: `32-48`|
|fileUid|string|yes|the file uid|
|force|string|no|default: `false`, Force deletion of non-empty subdirectories|

Let's delete the image we just created:

```go
package main

import (
	"fmt"
	"io/ioutil"
	libHttp "net/http"
	"strings"

	"github.com/bigfile/bigfile/databases/models"
	"github.com/bigfile/bigfile/http"
)

func main() {
	var (
		err         error
		request     *libHttp.Request
		token       = "49f92acd696260abba1bc4062d157199"
		tokenSecret = "9bcac735fd7f25947a3909998420affa"
	)
	params := map[string]interface{}{
		"token":   token,
		"nonce":   models.RandomWithMd5(255),
		"fileUid": "64c8a8ecd911630acf1dc26e8319f2dd",
	}
	body := http.GetParamsSignBody(params, tokenSecret)

	api := "https://127.0.0.1:10985/api/bigfile/file/delete?" + body
	if request, err = libHttp.NewRequest(libHttp.MethodDelete, api, strings.NewReader("")); err != nil {
		fmt.Println(err)
		return
	}
	resp, err := libHttp.DefaultClient.Do(request)
	if err != nil {
		fmt.Println(err)
		return
	}

	if bodyBytes, err := ioutil.ReadAll(resp.Body); err != nil {
		fmt.Println(err)
		return
	} else {
		fmt.Println(string(bodyBytes))
	}
}
```

Here we will get:

```json
{
    "requestId":10030,
    "success":true,
    "errors":null,
    "data":{
        "deletedAt":1567992964,
        "ext":"jpeg",
        "fileUid":"64c8a8ecd911630acf1dc26e8319f2dd",
        "hash":"9536467bde347627e27634a77963105a045f624e290b0f2bbc342834abdd4593",
        "hidden":0,
        "isDir":0,
        "path":"/images/png/profile/profile.jpeg",
        "size":4698744
    }
}
```

The field `deletedAt` exists and not null, indicates the has already been deleted. You can call the `/directory/list` to validate it. No accident, this will get:

```json
{
    "requestId":10031,
    "success":true,
    "errors":null,
    "data":{
        "items":[

        ],
        "pages":0,
        "total":0
    }
}
```