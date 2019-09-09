---
title: "RPC"
date: 2019-09-09T10:19:54+08:00
draft: false
---

To facilitate quick access to Bigfile in the application, we implemented the RPC service based on grpc. In theory, the languages ​​supported by [grpc](https://grpc.io/) can generate corresponding clients based on proto files, and quickly call Bigfile RPC services. Not much nonsense, let's start using it.

### Start RPC Server

In the RPC service, we support double-ended authentication, and we strongly recommend that you use it. You can generate certificates through the command line tools we provide.

    bigfile rpc:make-cert --server-cert-ips 127.0.0.1  --server-cert-ips 192.168.0.103

This command will output 6 files, they are: `ca.pem`, `ca.key`, `server.pem`, `server.key`, `client.pem` and `client.key`.

Next, let's start the server and use it.

    bigfile rpc:start --server-cert server.pem --server-key server.key --ca-cert ca.pem

After successful startup, you will get a message, like this:

    [2019/09/09 10:47:35.090] 18057 INFO  bigfile rpc service listening on: tcp://[::]:10986

### Connect to Serevr

Before actually using, we will connect to Server first. In subsequent use, we will skip this part. You also can find more detailed examples [here](https://godoc.org/github.com/bigfile/bigfile/rpc#pkg-examples).

```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io/ioutil"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
)

func createConnection() (*grpc.ClientConn, error) {
	var (
		err           error
		conn          *grpc.ClientConn
		cert          tls.Certificate
		certPool      *x509.CertPool
		rootCertBytes []byte
	)
	if cert, err = tls.LoadX509KeyPair("client.pem", "client.key"); err != nil {
		return nil, err
	}

	certPool = x509.NewCertPool()
	if rootCertBytes, err = ioutil.ReadFile("ca.pem"); err != nil {
		return nil, err
	}

	if !certPool.AppendCertsFromPEM(rootCertBytes) {
		return nil, err
	}

	if conn, err = grpc.Dial("192.168.0.103:10986", grpc.WithTransportCredentials(credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		RootCAs:      certPool,
	}))); err != nil {
		return nil, err
	}
	return conn, err
}

func main() {
	var (
		err  error
		conn *grpc.ClientConn
	)

	if conn, err = createConnection(); err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()
}
```

### Token Create 

Using rpc will make the code simpler，the bellow `rpc` package is from `github.com/bigfile/bigfile/rpc`.

```go
func main() {
	var (
		err  error
		conn *grpc.ClientConn
	)

	if conn, err = createConnection(); err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()

	grpcClient := rpc.NewTokenCreateClient(conn)
	fmt.Println(grpcClient.TokenCreate(context.TODO(), &rpc.TokenCreateRequest{
		AppUid:    "42c4fcc1a620c9e97188f50b6f2ab199",
		AppSecret: "f8f2ae1fe4f70b788254dcc991a35558",
	}))
}
```

### Token Update

Because it is too simple, so don't explain. the `timestamp` package is from `github.com/golang/protobuf/ptypes/timestamp`.

```go
func main() {
	var (
		err  error
		conn *grpc.ClientConn
	)

	if conn, err = createConnection(); err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()

	grpcClient := rpc.NewTokenUpdateClient(conn)
	fmt.Println(grpcClient.TokenUpdate(context.TODO(), &rpc.TokenUpdateRequest{
		AppUid:    "42c4fcc1a620c9e97188f50b6f2ab199",
		AppSecret: "f8f2ae1fe4f70b788254dcc991a35558",
		Token:     "1088f321f3c04becf04ec315fc023e81",
		ExpiredAt: &timestamp.Timestamp{Seconds: time.Now().Add(1000 * time.Minute).Unix()},
	}))
}
```

### Token Delete

```go
func main() {
	var (
		err  error
		conn *grpc.ClientConn
	)

	if conn, err = createConnection(); err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()

	grpcClient := rpc.NewTokenDeleteClient(conn)
	fmt.Println(grpcClient.TokenDelete(context.TODO(), &rpc.TokenDeleteRequest{
		AppUid:    "42c4fcc1a620c9e97188f50b6f2ab199",
		AppSecret: "f8f2ae1fe4f70b788254dcc991a35558",
		Token:     "1088f321f3c04becf04ec315fc023e81",
	}))
}
```

### File Create

Uploading the file is a bit complicated, let's do an explaination. In *File Create*, the server receives a streaming request and gets a response for each request. Too much nonsense is not as good as writing an example.

```go
package main

import (
	"context"
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io"
	"io/ioutil"
	"os"

	"github.com/bigfile/bigfile/databases/models"
	"github.com/bigfile/bigfile/rpc"
	"github.com/golang/protobuf/ptypes/wrappers"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
)

func createConnection() (*grpc.ClientConn, error) {
	var (
		err           error
		conn          *grpc.ClientConn
		cert          tls.Certificate
		certPool      *x509.CertPool
		rootCertBytes []byte
	)
	if cert, err = tls.LoadX509KeyPair("client.pem", "client.key"); err != nil {
		return nil, err
	}

	certPool = x509.NewCertPool()
	if rootCertBytes, err = ioutil.ReadFile("ca.pem"); err != nil {
		return nil, err
	}

	if !certPool.AppendCertsFromPEM(rootCertBytes) {
		return nil, err
	}

	if conn, err = grpc.Dial("192.168.0.103:10986", grpc.WithTransportCredentials(credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		RootCAs:      certPool,
	}))); err != nil {
		return nil, err
	}
	return conn, err
}

func main() {
	var (
		ctx            context.Context
		err            error
		resp           *rpc.FileCreateResponse
		cancel         context.CancelFunc
		client         rpc.FileCreateClient
		streamClient   rpc.FileCreate_FileCreateClient
		waitUploadFile *os.File
		conn           *grpc.ClientConn
	)

	if conn, err = createConnection(); err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()
	ctx, cancel = context.WithCancel(context.Background())
	defer cancel()
	client = rpc.NewFileCreateClient(conn)
	if streamClient, err = client.FileCreate(ctx); err != nil {
		fmt.Println(err)
		return
	}
	if waitUploadFile, err = os.Open("/Users/fudenglong/Downloads/profile.jpeg"); err != nil {
		fmt.Println(err)
		return
	}
	defer waitUploadFile.Close()
	for index := 0; ; index++ {
		var chunk = make([]byte, models.ChunkSize*2)
		var size int
		var quit bool
		if size, err = waitUploadFile.Read(chunk); err != nil {
			if err != io.EOF {
				fmt.Println(err)
				return
			}
			quit = true
		}
		req := &rpc.FileCreateRequest{
			Token:   "ee3caab522b1848744c9df3aa980346f",
			Path:    "/my-profiles/profile.jpeg",
			Content: &wrappers.BytesValue{Value: chunk[:size]},
		}
		if index == 0 {
			req.Operation = &rpc.FileCreateRequest_Overwrite{Overwrite: true}
		} else {
			req.Operation = &rpc.FileCreateRequest_Append{Append: true}
		}
		fmt.Println("sending request")
		if err = streamClient.Send(req); err != nil {
			fmt.Println(err)
			return
		}
		fmt.Println("waiting resp")
		if resp, err = streamClient.Recv(); err != nil {
			fmt.Println(err)
			return
		}
		fmt.Println(resp)
		if quit {
			break
		}
	}
}
```

### File Read

In *File Read*, you will get a stream response for file content.

```go
package main

import (
	"bytes"
	"context"
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io"
	"io/ioutil"
	"strconv"

	"github.com/bigfile/bigfile/internal/util"
	"github.com/bigfile/bigfile/rpc"
	"google.golang.org/grpc/metadata"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
)

func createConnection() (*grpc.ClientConn, error) {
	var (
		err           error
		conn          *grpc.ClientConn
		cert          tls.Certificate
		certPool      *x509.CertPool
		rootCertBytes []byte
	)
	if cert, err = tls.LoadX509KeyPair("client.pem", "client.key"); err != nil {
		return nil, err
	}

	certPool = x509.NewCertPool()
	if rootCertBytes, err = ioutil.ReadFile("ca.pem"); err != nil {
		return nil, err
	}

	if !certPool.AppendCertsFromPEM(rootCertBytes) {
		return nil, err
	}

	if conn, err = grpc.Dial("192.168.0.103:10986", grpc.WithTransportCredentials(credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		RootCAs:      certPool,
	}))); err != nil {
		return nil, err
	}
	return conn, err
}

func main() {
	var (
		err  error
		conn *grpc.ClientConn
	)

	if conn, err = createConnection(); err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()

	var (
		client       = rpc.NewFileReadClient(conn)
		header       metadata.MD
		fileName     string
		fileHash     string
		dataHash     string
		dataBuffer   *bytes.Buffer
		fileSize     int
		streamClient rpc.FileRead_FileReadClient
	)
	if streamClient, err = client.FileRead(context.Background(), &rpc.FileReadRequest{
		Token:   "ee3caab522b1848744c9df3aa980346f",
		FileUid: "b58a51fc0ee8d95918c9715bfdb2af0d",
	}); err != nil {
		fmt.Println(err)
		return
	}
	if header, err = streamClient.Header(); err != nil {
		fmt.Println(err)
		return
	}
	fileName = header.Get("name")[0]
	fileHash = header.Get("hash")[0]
	if fileSize, err = strconv.Atoi(header.Get("size")[0]); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("name = %s, hash = %s, size = %d\n", fileName, fileHash, fileSize)
	dataBuffer = new(bytes.Buffer)
	for {
		if resp, err := streamClient.Recv(); err != nil {
			if err != io.EOF {
				fmt.Println(err)
				return
			}
			break
		} else {
			if _, err = dataBuffer.Write(resp.Content); err != nil {
				fmt.Println(err)
				return
			}
		}
	}
	if dataHash, err = util.Sha256Hash2String(dataBuffer.Bytes()); err != nil {
		fmt.Println(err)
		return
	}
	if dataHash != fileHash {
		fmt.Println("file is broken")
		return
	}

	// here, you should put fileContent to a file, example:
	// _ = ioutil.WriteFile(fileName, dataBuffer.Bytes(), 0666)
}
```

### File Update

I know that you are the smartest, so I don't waste time.

```go
func main() {
	var (
		err  error
		conn *grpc.ClientConn
	)
	if conn, err = createConnection(); err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()
	c := rpc.NewFileUpdateClient(conn)
	resp, err := c.FileUpdate(context.Background(), &rpc.FileUpdateRequest{
		Token:   "ee3caab522b1848744c9df3aa980346f",
		FileUid: "b58a51fc0ee8d95918c9715bfdb2af0d",
		Path:    "/another/dir/profile.jpeg",
	})
	fmt.Println(resp, err)
}
```

### Directory List

`wrappers` package is from `github.com/golang/protobuf/ptypes/wrappers`.

```go
func main() {
	var (
		err  error
		conn *grpc.ClientConn
	)
	if conn, err = createConnection(); err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()
	c := rpc.NewDirectoryListClient(conn)
	resp, err := c.DirectoryList(context.Background(), &rpc.DirectoryListRequest{
		Token:  "ee3caab522b1848744c9df3aa980346f",
		Sort:   rpc.DirectoryListRequest_AscType,
		SubDir: &wrappers.StringValue{Value: "/another/dir"},
	})
	fmt.Println(resp, err)
}
```

### File Delete

It’s so simple, I don’t know what to say.

```go
func main() {
	var (
		err  error
		conn *grpc.ClientConn
	)
	if conn, err = createConnection(); err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()
	c := rpc.NewFileDeleteClient(conn)
	resp, err := c.FileDelete(context.Background(), &rpc.FileDeleteRequest{
		Token:            "ee3caab522b1848744c9df3aa980346f",
		FileUid:          "b58a51fc0ee8d95918c9715bfdb2af0d",
		ForceDeleteIfDir: false,
	})
	fmt.Println(resp, err)
}
```

In the above example, we only give examples of the Go language. Other languages ​​will be given in the following steps. In fact, you only need to generate the corresponding language client based on the protos file we have written. such as for go:	

	protoc -I protos  protos/* --go_out=plugins=grpc,paths=source_relative:.

Documents for other languages you can find [here](https://grpc.io/docs/quickstart/).