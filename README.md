ssh2http [![Build Status](https://travis-ci.org/mdlayher/ssh2http.svg?branch=master)](https://travis-ci.org/mdlayher/ssh2http) [![GoDoc](http://godoc.org/github.com/mdlayher/ssh2http?status.svg)](http://godoc.org/github.com/mdlayher/ssh2http)
======

Package `ssh2http` provides functionality that enables some functionality of Go's
`net/http` package to be used with SSH servers using SFTP.  MIT Licensed.

Examples
========

Currently, `ssh2http` provides two types which can be used with `net/http`.

FileSystem
----------

`ssh2http.FileSystem` can be used as a `http.FileSystem` which enables a user to
browse and access files on a remote machine, using a local HTTP server.

```go
package main

import (
	"log"
	"net/http"

	"github.com/IoTServ/ssh2http"
	"golang.org/x/crypto/ssh"
)

func main() {
	// Set up a http.FileSystem pointed at a user's home directory on
	// a remote server.
	fs, err := ssh2http.NewFileSystem("sftp://192.168.1.1:22/home/user", &ssh.ClientConfig{
		User: "user",
		Auth: []ssh.AuthMethod{
			ssh.Password("password"),
		},
	})
	if err != nil {
		log.Fatal(err)
	}

	// Bind HTTP server, provide link for user to browse files
	host := ":8080"
	log.Printf("starting listener: http://localhost%s/", host)
	if err := http.ListenAndServe(":8080", http.FileServer(fs)); err != nil {
		log.Fatal(err)
	}
}
```

RoundTripper
------------

`ssh2http.RoundTripper` can be used as a `http.RoundTripper` which enables a user
to use Go's `net/http` library directly over SSH with SFTP.  In the future, this
method will support adding, updating, and deleting remote files.

```go
package main

import (
	"io"
	"log"
	"net/http"
	"os"

	"github.com/IoTServ/ssh2http"
	"golang.org/x/crypto/ssh"
)

func main() {
	log.SetOutput(os.Stderr)

	// Set up a ssh2http.RoundTripper with a default set of credentials.  These
	// credentials will be used whenever a SSH host is dialed, unless another
	// set is provided directly via the RoundTripper.Dial method.
	rt := ssh2http.NewRoundTripper(&ssh.ClientConfig{
		User: "user",
		Auth: []ssh.AuthMethod{
			ssh.Password("password"),
		},
	})

	/*
		// If needed, more hosts can be dialed with different credentials.
		// The ssh2http.RoundTripper will automatically use these credentials
		// for future interactions with this host.
		if err := rt.Dial("192.168.1.2:22", &ssh.ClientConfig{
			User: "user2",
			Auth: []ssh.AuthMethod{
				ssh.Password("password2"),
			},
		}); err != nil {
			log.Fatal(err)
		}
	*/

	// Set up a http.Client with ssh2http.RoundTripper registered to handle SFTP
	// URLs.
	t := &http.Transport{}
	t.RegisterProtocol(ssh2http.Protocol, rt)
	c := &http.Client{Transport: t}

	// Perform a HTTP GET request over SFTP, to download a file.
	res, err := c.Get("sftp://192.168.1.1:22/home/user/hello.txt")
	if err != nil {
		log.Fatal(err)
	}

	// Copy file directly to stdout.
	if _, err := io.Copy(os.Stdout, res.Body); err != nil {
		log.Fatal(err)
	}

	// Close all open ssh2http.RoundTripper connections.
	if err := rt.Close(); err != nil {
		log.Fatal(err)
	}
}
```
