--- 
layout: post
title: Transfer-Encoding的作用
comments: true
categories:
- web
- http
status: publish
type: post
published: true
meta: 
  _edit_last: "1"
---

通过HTTP传送数据时，有些时候并不能事先确定body的长度，因此无法得到[Content-Length][]的值，
就不能在header中指定Content-Length了，造成的最直接的影响就是：接收方无法通过Content-Length得到报文体的长度，
那怎么判断发送方发送完毕了呢？HTTP
1.1协议在header中引入了[Transfer-Encoding][]，当其值为chunked时,
表明采用chunked编码方式来进行报文体的传输。chunked编码的基本方法是将大块数据分解成多块小数据，每块都可以自指定长度，
其格式如下:

> If a Transfer-Encoding field with a value of chunked is specified in
> an HTTP message (either a request sent by a client or the response
> from the server), the body of the message consists of an unspecified
> number of chunks, a terminating last-chunk, an optional trailer of
> entity header fields, and a final CRLF sequence.

> Each chunk starts with the number of octets of the data it embeds
> expressed in hexadecimal followed by optional parameters (chunk
> extension) and a terminating CRLF (carriage return and line feed)
> sequence, followed by the chunk data. The chunk is terminated by CRLF.
> If chunk extensions are provided, the chunk size is terminated by a
> semicolon followed with the extension name and an optional equal sign
> and value.

> The last chunk is a zero-length chunk, with the chunk size coded as 0,
> but without any chunk data section. The final chunk may be followed by
> an optional trailer of additional entity header fields that are
> normally delivered in the HTTP header to allow the delivery of data
> that can only be computed after all chunk data has been generated. The
> sender may indicate in a Trailer header field which additional fields
> it will send in the trailer after the chunks.

但凡web server支持 HTTP
1.1，就应该支持Transfer-Encoding的传送方式。apache当然也支持这种传送方式。
简简单单写个程序验证下。

服务器端，一个cgi(mirror.cgi)，将获取的标准输入直接输出到标准输出即可。也就是说将从客户端获得的报文体又作为报文体返回给客户端。
这样来验证客户端通过Transfer-Encoding传送，是否达到预想的目的。

~~~~ python
#!/usr/bin/env python

import sys

BUFFER_SIZE = 1024

sys.stdout.write("Content-type: text/html\n\n")
while True:
    buffer = sys.stdin.read(BUFFER_SIZE)
    sys.stdout.write(buffer)

    if len(buffer) != BUFFER_SIZE:
        break
~~~~

客户端，按照Transfer-Encoding为chunked的format，来传递数据。比如我们想传递一个文件名为file的文件内容
作为报文体的内容传送给服务端。由于file的内容比较大，一下子传递，内存估计吃不消，就可以采用分批传送。

~~~~ python
#!/usr/bin/env python

import httplib

conn = httplib.HTTPConnection("127.0.0.1")
conn.putrequest("PUT", "/cgi-bin/mirror.cgi")
conn.putheader("Transfer-Encoding", "chunked")
conn.endheaders()

with open("file") as fp:
    for line in fp.readlines():
        conn.send("%x" % len(line) + "\r\n" + line + "\r\n")

conn.send("0\r\n\r\n")

response = conn.getresponse()
print response.read()
~~~~

References & Resources:

-   [Chunked transfer encoding][]
-   [RFC2616 Transfer-Encoding][Transfer-Encoding]
-   [RFC2616 Transfer-Codings][]
-   [RFC2616 Content-Length][Content-Length]

  [Content-Length]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.13
  [Transfer-Encoding]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.41
  [Chunked transfer encoding]: http://en.wikipedia.org/wiki/Chunked_transfer_encoding
  [RFC2616 Transfer-Codings]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.6
