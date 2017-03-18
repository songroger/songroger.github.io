---
layout: post
title:  "Regular use of Mitmproxy"
date:   2017-02-21 12:15:14 +0000
categories: self words
tail: <a href="http://docs.mitmproxy.org/en/v0.17/mitmproxy.html#example-interception">[More]</a>
description: 'Knowing about this is enough'
---
### Install the certificate
Just start mitmproxy and configure your target device with the correct proxy settings. Now start a browser on the device, and visit the magic domain **[mitm.it](http://mitm.it)**. 
Click on the relevant icon to install the certificate.

*Note: You should go to the system settings->security->certificates to import the ca. for some android devices.*

### Interception
Mitmproxy's interception functionality lets you pause an HTTP request or response, inspect and modify it, and then accept it to send it on to the server or client.

1. `>>> mitmproxy` 
2. Press `i` to set an interception filter.eg "~q". 
Intercepted connections are indicated with **orange** text
3. View and modify the request:
viewed the request by selecting it, pressed `e` for “edit”
4. Press `a` to accept the modified request, which is then sent on to the server


### Inline Scripts
Use your own scripts to get the response.

examples/test.py
```python
def response(context, flow):
    flow.response.headers["newheader"] = "foo"
    if flow.request.host == "eg.com":
        flow.request.scheme = "http"
        flow.request.port = 8080
        flow.request.path = flow.request.path.replace(
            "v1/", "v2/")
```

1. `>>> mitmdump -s test.py` 
2. `>>> mitmproxy -s test.py`
