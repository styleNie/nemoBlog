#Python 发送手机短信

```python 
#!/usr/local/bin/python
#-*- coding:utf-8 -*-
import httplib
import urllib
 
host  = "106.ihuyi.com"
sms_send_uri = "/webservice/sms.php?method=Submit"
 
#用户名是登录ihuyi.com账号名（例如：cf_demo123）
account  = "用户名"
#密码 查看密码请登录用户中心->验证码、通知短信->帐户及签名设置->APIKEY
password = "密码"
 
def send_sms(text, mobile):
    params = urllib.urlencode({'account': account, 'password' : password, 'content': text, 'mobile':mobile,'format':'json' })
    headers = {"Content-type": "application/x-www-form-urlencoded", "Accept": "text/plain"}
    conn = httplib.HTTPConnection(host, port=80, timeout=30)
    conn.request("POST", sms_send_uri, params, headers)
    response = conn.getresponse()
    response_str = response.read()
    conn.close()
    print response.status
    print response.reason
    print response.read()
    return response_str

def sendMsg_sub(tos,content):
    content=content.decode("UTF-8")
    print(len(content))
    if len(content) <= 329:
        sendMessages(tos,content.encode("UTF-8"))
    else:
    # 内容太长，分条发送
        ran = int(round(len(content) *1.0/324))
        for i in range(1,ran+1):
            if i*324 > len(content):
                contenti = "["+ str(i) + "/" + str(ran) + "]"+content[(i-1)*324 : len(content)]
            else:
                contenti = "[" + str(i) + "/" + str(ran) + "]" + content[(i - 1) * 324: i*324]
            sendMessages(tos, contenti.encode("UTF-8"))
            time.sleep(10)
            
            
if __name__ == '__main__':
 
    mobile = "138xxxxxxxx"
    text = "您的验证码是：121254。请不要把验证码泄露给其他人。"
 
    print(send_sms(text, mobile))
```

Reference: [互亿无线](http://www.ihuyi.com/demo/sms/python.html)