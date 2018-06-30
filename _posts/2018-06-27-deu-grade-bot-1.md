---
layout: post
current: post
navigation: True
cover: assets/images/deu-grade-bot/deu-grade-logo.png
title: Telegram을 이용한 동의대학교 성적 알림 봇 만들기 - 1
tags: chatbot telegram python
class: post-template
subclass: 'post tag-chatbot'
author: qwertycvb
---

이번에 성적 알림봇을 만들어 본 이유는 성적 확정 기간까지 언제 어떤 과목의 성적이 올라올지 모르는 상황에서 수시로 앱에 들어가 확인하는것이 너무 귀찮았기 때문입니다(...) 자신이 필요한 것을 자급자족하는 참된 개발자가 됩시다.

![pic1](assets/images/deu-grade-bot/pic1.png)

이번 프로젝트를 통하여 사진과 같이 성적 변동을 알려주는 텔레그램 봇을 만들어 봅시다.

---

### 동의대 앱을 통한 로그인 URL 및 파라미터 확인

이 과정은 그리 어렵지 않았습니다.  휴대폰 내에서 패킷 캡쳐 앱을 설치해 패킷을 캡쳐하니 바로 제가 원하는 결과를 얻을 수 있었습니다.

![pic2](assets/images/deu-grade-bot/pic2.png)

패킷 캡쳐를 통해 동의대 앱에서는 `http://smartcampus.deu.ac.kr/auth/user/loginAuth` 라는 주소로 `URLENCODED` 방식을 통해 다양한 파라미터를 담아서 HTTP POST를 시도함을 알 수 있었습니다. 물론 그에 필요한 파라미터들도 함께 얻을 수 있었습니다.

파라미터에는 entryA - **암호화 된 ID(학번)**, entryB -  **암호화 된 PW**, strongbox, ip가 있네요.

> ip는 내용에 관계없이 파라미터가 전달 되면 됩니다. 따라서 본 게시글에서는 dummy값을 POST 합니다.

---

### 기초 코드 구성

위의 패킷 캡쳐를 통해 얻은 정보들을 통해 토대가 되는 코드를 구성해보았습니다.

{% highlight python %}
import requests

requests.post("http://smartcampus.deu.ac.kr/auth/user/loginAuth", data={'entryA': id_enc, 'entryB': pw_enc, 'strongbox': 'deusmartcampus', 'ip': 'dummy'})
{% endhighlight %}

~~~shell
<Response [200]>
~~~

HTTP RESPONSE CODE가 200 OK로 나왔습니다. 200의 의미는 POST를 통하여 성공적으로 data를 전송했다는 것이지 로그인에 성공했다는 의미는 아닙니다.

따라서 이번엔 Response Body를 확인할 수 있게끔 코드를 수정했습니다.

{% highlight python %}
import requests

res = requests.post("http://smartcampus.deu.ac.kr/auth/user/loginAuth", data={'entryA': id_enc, 'entryB': pw_enc, 'strongbox': 'deusmartcampus', 'ip': 'dummy'})

print(res.text)
{% endhighlight %}


{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<user>	   
	<result>00</result>
	<resultMessage>SUCCESS</resultMessage>
	<resultRowCount>1</resultRowCount>       	
	<suser_id>2018xxxx</suser_id>
	<suser_nm>진희륜</suser_nm>
	<dept_nm>컴퓨터공학과</dept_nm>
	<dept_cd></dept_cd>
	<sts></sts>
	<emp_div_cd></emp_div_cd>
	<dept_div></dept_div>
	<emp_div></emp_div>
</user>
{% endhighlight %}

다행히도 로그인이 성공적으로 진행되어 다양한 정보를 XML형식으로 Response 받았음을 확인할 수 있습니다.

여기까지만(로그인 과정) 이루어져도 크게 상관은 없지만 저는 스크립트 실행시 누군지 판별할 수 있도록 BeautifulSoup를 통해 XML을 파싱하였습니다.

---

### BeautifulSoup를 이용한 XML 파싱

{% highlight python linenos %}
import requests

res = requests.post("http://smartcampus.deu.ac.kr/auth/user/loginAuth", data={'entryA': id_enc, 'entryB': pw_enc, 'strongbox': 'deusmartcampus', 'ip': 'dummy'})

res = BeautifulSoup(res.text, "lxml")
name = res.findAll("suser_nm")
id = res.findAll("suser_id")
dept = res.findAll("dept_nm")

print("이름 : " + name[0].text + "\n학번 : " + id[0].text + "\n학과 : " + dept[0].text + "\n")
{% endhighlight %}

~~~
이름 : 진희륜
학번 : 2018xxxx
학과 : 컴퓨터공학과
~~~

XML 파싱이 제대로 이루어져 로그인 체크 겸 사용자를 확인할 수 있네요.



다음편에 계속됩니다.