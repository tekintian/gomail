# Gomail
[![Build Status](https://travis-ci.org/go-gomail/gomail.svg?branch=v2)](https://travis-ci.org/go-gomail/gomail) [![Code Coverage](http://gocover.io/_badge/gopkg.in/gomail.v2)](http://gocover.io/gopkg.in/gomail.v2) [![Documentation](https://godoc.org/gopkg.in/gomail.v2?status.svg)](https://godoc.org/gopkg.in/gomail.v2)

## Introduction

Gomail is a simple and efficient package to send emails. It is well tested and
documented.

Gomail can only send emails using an SMTP server. But the API is flexible and it
is easy to implement other methods for sending emails using a local Postfix, an
API, etc.

It is versioned using [gopkg.in](https://gopkg.in) so I promise
there will never be backward incompatible changes within each version.

It requires Go 1.2 or newer. With Go 1.5, no external dependencies are used.


## Features

Gomail supports:
- Attachments
- Embedded images
- HTML and text templates
- Automatic encoding of special characters
- SSL and TLS
- Sending multiple emails with the same SMTP connection


## Documentation

https://godoc.org/gopkg.in/gomail.v2


## Download

    go get gopkg.in/gomail.v2


## Examples

See the [examples in the documentation](https://godoc.org/gopkg.in/gomail.v2#example-package).

~~~go
package main

import (
	"fmt"
	"log"
	"time"

	"gopkg.in/gomail.v2"
)

// The list of recipients.
var list []struct {
	Name    string
	Address string
}

func main() {
	// 创建一个管道ch 大小为 1
	ch := make(chan *gomail.Message, 1)
	chExit := make(chan int, 1)

	// 准备要发送的信息
	m := gomail.NewMessage()
	// 单条信息
	m.SetHeader("From", "alex@example.com")
	m.SetHeader("To", "bob@example.com", "cora@example.com")
	m.SetAddressHeader("Cc", "dan@example.com", "Dan")
	m.SetHeader("Subject", "Hello!")
	m.SetBody("text/html", "Hello <b>Bob</b> and <i>Cora</i>!")
	m.Attach("/home/Alex/lolcat.jpg")
	// 将要发送的信息放入管道
	ch <- m

	// 批量发送 这里的 list 是一个包含接收邮件的对象列表
	// for _, r := range list {
	// 	m.SetHeader("From", "no-reply@example.com")
	// 	m.SetAddressHeader("To", r.Address, r.Name)
	// 	m.SetHeader("Subject", "Newsletter #1")
	// 	m.SetBody("text/html", fmt.Sprintf("Hello %s!", r.Name))

	// 	if err := gomail.Send(s, m); err != nil {
	// 		log.Printf("Could not send email to %q: %v", r.Address, err)
	// 	}
	// 	m.Reset()//重置发送信息
	// }

	// 关闭管道以停止邮件发送协程
	close(ch)

	go func(ch chan *gomail.Message, chExit chan int) {
		// NewDialer returns a new SMTP Dialer. The given parameters are used to connect to the SMTP server.
		dialer := gomail.NewDialer("smtp.qq.com", 587, "user", "123456")
		// 变量定义
		var (
			sendCloser gomail.SendCloser // 这个是用来接收 dialer.Dial()返回的 SendCloser 注意这是一个接口,用于接收实现了这个接口的对象
			err        error
			open       = false // smtp链接是否就绪的标志定义
			tchan = time.After(5 * time.Second)
		)
		for {
			select {
			// 这里的管道读取 <-ch 会阻塞当前线程
			case m, ok := <-ch:
				if !ok { // 如果管道读取异常,则直接返回 结束运行
					fmt.Println("退出协程")
					chExit <- 1
					return
				}
				if !open {
					// Dial dials and authenticates to an SMTP server. The returned SendCloser should be closed when done using it.
					if sendCloser, err = dialer.Dial(); err != nil {
						fmt.Println("Smtp认证失败!", err) // smtp登录认证异常
						chExit <- 1
						return
					}
					open = true
				}
				// 发送邮件
				if err := gomail.Send(sendCloser, m); err != nil {
					log.Print(err)
				} else {
					fmt.Println("Email send successfully!")
				}
			// 30秒后没有邮件发送,则关闭smtp链接
			case <-tchan:
				if open {
					// 关闭smtp
					if err := sendCloser.Close(); err != nil {
						fmt.Println("超时关闭")
						chExit <- 1
						return
					}
					open = false
				}
			}
		}
	}(ch, chExit)

	for {
		if _, ok := <-chExit; ok {
			break // 协程退出,这里需要终止这里的for循环
		}
	}

	fmt.Println("end....")
}

~~~

## FAQ

### x509: certificate signed by unknown authority

If you get this error it means the certificate used by the SMTP server is not
considered valid by the client running Gomail. As a quick workaround you can
bypass the verification of the server's certificate chain and host name by using
`SetTLSConfig`:

    package main

    import (
    	"crypto/tls"

    	"gopkg.in/gomail.v2"
    )

    func main() {
    	d := gomail.NewDialer("smtp.example.com", 587, "user", "123456")
    	d.TLSConfig = &tls.Config{InsecureSkipVerify: true}

        // Send emails using d.
    }

Note, however, that this is insecure and should not be used in production.


## Contribute

Contributions are more than welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for
more info.


## Change log

See [CHANGELOG.md](CHANGELOG.md).


## License

[MIT](LICENSE)


## Contact

You can ask questions on the [Gomail
thread](https://groups.google.com/d/topic/golang-nuts/jMxZHzvvEVg/discussion)
in the Go mailing-list.
