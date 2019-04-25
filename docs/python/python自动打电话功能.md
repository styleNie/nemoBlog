---
title: "python自动打电话功能"    
author:     
date: April 04, 2018     

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: false    
  ignoreLink:false    
  
html:    
  embed_local_images: true    
  embed_svg: true    
  offline: false    
  toc: false    

print_background: false    

export_on_save:    
  html: true    

---

# <center>python自动打电话功能</center>
 

# 自动打电话   
功能实现是由[twilio](https://www.twilio.com)提供的，先看代码。


python版   
```python 
from twilio.rest import Client

account = "AC0827435504ed63c3af999702df7d065d"
token = "your_auth_token"
client = Client(account, token)

call = client.calls.create(to="+15558675310",
                           from_="+15017122661",
                           url="https://demo.twilio.com/welcome/voice/")
print(call.sid)
```

java版   
```java 
import java.net.URI;
import java.net.URISyntaxException;

import com.twilio.Twilio;
import com.twilio.rest.api.v2010.account.Call;
import com.twilio.type.PhoneNumber;

public class MakePhoneCall {
    // Find your Account Sid and Token at twilio.com/console
    public static final String ACCOUNT_SID = "AC0827435504ed63c3af999702df7d065d";
    public static final String AUTH_TOKEN = "your_auth_token";

    public static void main(String[] args) throws URISyntaxException {
        Twilio.init(ACCOUNT_SID, AUTH_TOKEN);

        String from = "+15017122661";
        String to = "+15558675310";

        Call call = Call.creator(new PhoneNumber(to), new PhoneNumber(from),
                new URI("https://demo.twilio.com/welcome/voice/")).create();

        System.out.println(call.getSid());
    }
}
```   

java依赖的包   
```
<dependency>
    <groupId>com.twilio.sdk</groupId>
        <artifactId>twilio</artifactId>
    <version>7.17.0</version>
</dependency>
```   

上面的代码中都需要Account和token，可以在[twilio](https://www.twilio.com) 注册一个账号，获取Account和token，注册成功后账户里面有赠送的15美元；然后在[Phone Numbers](https://www.twilio.com/console/phone-numbers/incoming)页面获取免费的打电话的号码，也就是上面代码中的**from**,接下来需要验证你自己的电话号码可以打通，需要在[Voice Geographic Permissions](http://twilio.com/console/voice/settings/geo-permissions)或者[Verified Caller IDs](https://www.twilio.com/console/phone-numbers/verified)页面添加受信任的电话号码，这里的号码都是遵循E164格式的国际号码；运行上面的代码就可以接到电话了。