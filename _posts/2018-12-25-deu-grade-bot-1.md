---
layout: post
current: post
navigation: True
cover: assets/images/deu-grade-bot/deu-grade-logo.png
title: Python과 Telegram을 이용한 동의대 성적 알림 봇 만들기 - 1
tags: chatbot gradebot python
class: post-template
subclass: 'post tag-chatbot'
author: qwertycvb
---

성적 확인 봇을 만드는 과정은 여러 절로 구성됩니다.

1. [성적 관리 서비스에 로그인](https://blog.qwertycvb.com/deu-grade-bot-1)
2. [정보를 가공하고 메신저로 보내기](https://blog.qwertycvb.com/deu-grade-bot-2)
3. 암호화 된 ID와 PW 구하기 (번외)

이 글은 1번 글입니다.

또한 소스는 제 [Github Repo](https://github.com/SerenityS/deugradebot)에 공개 되어있습니다.

---

### 만들고자 하는 것

![result](assets/images/deu-grade-bot/result.png)

와 같이 작동하는 챗봇 알리미

## 작업 전 필요한 것들

1. Python 3.x
2. Python IDE
3. 서버 (라즈베리 파이나 VPS 등)

#### 봇 동작에 필요한 파이썬 모듈 설치

``pip3 install bs4 lxml requests python-telegram-bot``

---
## 동의대 성적 관리 서비스에 로그인

동의대 앱에서 패킷 캡쳐를 해보면 동의대 성적 관리 서비스에 접근하기 위해서는 세번의 로그인이 필요함을 알 수 있습니다.

![pcap_1](assets/images/deu-grade-bot/pcap_1.png)

1.JBoss 서비스에 로그인 (Encrpyt된 ID와 PW 요구)

![pcap_2](assets/images/deu-grade-bot/pcap_2.png)

2.Josso 서비스에 로그인 (평문 ID와 Encrpyt된 PW 요구)

![pcap_3](assets/images/deu-grade-bot/pcap_3.png)

3.성적 확인 사이트에 로그인 (2번 과정에서 얻은 쿠키를 이용)

이때 암호화된 ID와 PW는 아직은 암호화 방식을 알 수 없기에 패킷 캡쳐를 통해 얻은 암호화된 ID와 PW를 이용하여 진행하겠습니다.

---
#### Session 구현

파이썬을 통해 웹 파싱을 할때 로그인 등의 동작이 필요하고, 이 로그인 데이터의 보존이 필요할 때 파이썬에서는 세션의구현을 통해 로그인 데이터의 보존이 가능합니다.

{% highlight python %}
import requests

with requests.Session() as s:
​     s.post(~~)

{% endhighlight %}

이는 위와 같은 간단한 코드로 구현이 가능합니다. 이때는 ``with requests.Session() as s:`` 블럭에서만 세션이 유지됩니다.

---
#### JBoss 서비스에 로그인

![jboss_req](assets/images/deu-grade-bot/jboss_req.png)

패킷 캡쳐를 통해 얻은 정보로 로그인 과정을 구성해보면

``http://smartcampus.deu.ac.kr/auth/user/loginAuth`` 에 request param을 담아서 post를 시도합니다.

이때 Request Parameter로 ``entryA ``- 암호화 된 ID, ``entryB`` - 암호화 된 PW, ``strongbox`` - deusmartcampus, 그리고 ``ip``를 요구함을 알 수 있습니다.

이를 파이썬 코드로 구성해보면

{% highlight python %}
import requests

with requests.Session() as s:
​    s.post("http://smartcampus.deu.ac.kr/auth/user/loginAuth", data={'entryA': id_enc, 'entryB': pw_enc, 'strongbox': 'deusmartcampus'})
​    print(res.text)
{% endhighlight %}
와 같이 구성할 수 있습니다. IP는 전달해주지 않아도 상관 없어서 제외했습니다.

이 post에 대한 response body의 데이터(res.text)를 확인해보면

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<user>	   
​	<result>00</result>
​	<resultMessage>SUCCESS</resultMessage>
​	<resultRowCount>1</resultRowCount>       	
​	<suser_id>2018xxxx</suser_id>
​	<suser_nm>진희륜</suser_nm>
​	<dept_nm>컴퓨터공학과</dept_nm>
​	<dept_cd></dept_cd>
​	<sts></sts>
​	<emp_div_cd></emp_div_cd>
​	<dept_div></dept_div>
​	<emp_div></emp_div>
</user>
{% endhighlight %}

와 같이 패킷 캡쳐에서 봤던 response body를 정상적으로 획득했음을 알 수 있습니다.

---
#### Josso에 로그인

Josso 로그인 과정도 앞서 본 JBoss 로그인 과정과 같습니다.

![josso_req](assets/images/deu-grade-bot/josso_req.png)

``http://smartcampus.deu.ac.kr/josso/signon/usernamePasswordLogin.do``에 로그인 하는데, 이 JBoss에 로그인 할때도 다양한 Parameter를 요구하고 있지만 실제로 필요한 Parameter은  ``josso_cmd`` - login, ``josso_username`` - 평문 ID, ``josso_password`` - 암호화된 PW 입니다.

이를 앞선 JBoss 로그인을 포함하여 파이썬 코드로 구성하면 아래와 같습니다.

{% highlight python %}
import requests

with requests.Session() as s:
​    s.post("http://smartcampus.deu.ac.kr/auth/user/loginAuth", data={'entryA': id_enc, 'entryB': pw_enc, 'strongbox': 'deusmartcampus'})
​    josso = s.post('http://smartcampus.deu.ac.kr/josso/signon/usernamePasswordLogin.do', data={'josso_cmd': 'login', 'josso_username': id, 'josso_password': enc_pw})
​    print(josso.cookies)
{% endhighlight %}

그리고 결과 값으로

{% highlight shell %}
<RequestsCookieJar[<Cookie JOSSO_SESSIONID_josso=~cookie~ for smartcampus.deu.ac.kr/josso>, <Cookie JSESSIONID=~cookie~ for smartcampus.deu.ac.kr/josso>]>
{% endhighlight %}

처럼 쿠키를 얻을 수 있음을 알 수 있습니다. 이 쿠키는 세번째 로그인 과정인 성적 확인 시스템에 로그인할 때 사용됩니다.

---
#### 성적 확인 사이트에 로그인

성적 확인 사이트는 앞서 2번과정에서 얻은 쿠키를 이용하여 로그인합니다.

![info_req](assets/images/deu-grade-bot/info_req.png)

이번에 앞선 과정과 조금 다른 부분은 ``http://smartcampus.deu.ac.kr/webservice/302/D/01/01``에 JOSSO 쿠키를 가지고 GET요청을 한다는 것 뿐입니다.

{% highlight python %}
import requests

with requests.Session() as s:
​    s.post("http://smartcampus.deu.ac.kr/auth/user/loginAuth", data={'entryA': id_enc, 'entryB': pw_enc, 'strongbox': 'deusmartcampus'})
​    josso = s.post('http://smartcampus.deu.ac.kr/josso/signon/usernamePasswordLogin.do', data={'josso_cmd': 'login', 'josso_username': id, 'josso_password': enc_pw})
​    res = s.get('http://smartcampus.deu.ac.kr/webservice/302/D/01/01', cookies=josso.cookies)
​    print(res.text)
{% endhighlight %}

자세히 보시면 아시겠지만 마지막만 s.get 명령을 통해 GET하는 것을 확인하실 수 있습니다. 또한 cookies= 를 통해 cookie를 전달해줍니다.

너무 길어서 생략했지만 res.text의 값은 성적 정보를 포함한 아래 사진의 레이아웃을 구성하고 있는 html 파일임을 확인할 수 있습니다.

![deu_grade_example](assets/images/deu-grade-bot/deu_grade_example.png)