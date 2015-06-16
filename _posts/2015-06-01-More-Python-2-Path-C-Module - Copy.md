---
layout: post

title: More Python 2 - Patch C Module
cover_image: blog-cover.png

excerpt: "C 모듈을 패치 해보자!"

author:
  name: 박아윤
  bio: Software Engineer
  image: pay.jpg
  page: http://www.parkayun.kr
---

안녕하세요, [애드투페이퍼](http://www.add2paper.com)의 [박아윤](http://www.parkayun.kr)입니다. <br />
[첫 번째 More Python](/2014/12/01/More-Python-1-Copy/)을 작성하고 정확히 반년 뒤, 의병의 날인 오늘 그 두 번째 글을 작성하게 되었습니다. 하하<br />

##Patch
이번에 다룰 내용은 흔히들 Monkey Patch라고들 부르는 Patch에 관한 이야기입니다.<br />
Python에서는 이러한 Patch를 아래와 같이 할 수 있습니다.

{% highlight python %}
>>> import django
>>> def patch_django():
...    return "Flask!"
>>> django = patch_django
>>> django()
'Flask!'
{% endhighlight %}

이렇게 Python의 은혜를 받아 매우 손쉽게 Patch를 할 수 있습니다. <br />
이런 은혜에도 불구하고 문제점이 하나 있는데요, 아래와 같이 코드를 보면 Patch가 원활히 되지 않습니다. :(

{% highlight python %}
>>> from datetime import date
>>> def patch_today():
...    return date(2015, 1, 1)
>>> date.today = patch_today()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: can't set attributes of built-in/extension type 'datetime.date'
{% endhighlight %}

datetime.date의 속성을 지정할 수 없다고 합니다. (C 모듈은 Monkey Patch를 할 수 없습니다.)<br />
일단 영문을 모르겠으니 묻지도 따지지도 않고 파일을 찾아 디버깅을 해보겠습니다.

{% highlight python %}
>>> datetime.__file__
'/Somewhere/lib/python2.7/lib-dynload/datetime.so'
{% endhighlight %}

아니! Python 파일이 아닙니다. C로 작성된 모듈 같은데 이제 어쩌죠? (저는 Python밖에 모른다고요...)<br /><del>이제 이만 글을 마칩니다. 읽어주셔서 감사합니다.</del>

하늘이 무너져도 솟아날 구멍이 있다더니 ctypes 모듈을 이용하면 이러한 c로 작성된 모듈이나 외부 라이브러리를 다룰 수 있습니다. 그러면 ctypes를 이용해 dir와 비슷해 보이는, 속성들을 가져오는 get_dict를 작성하겠습니다.

{% highlight python %}
>>> import ctypes
>>> _get_dict = ctypes.pythonapi._PyObject_GetDictPtr
>>> _get_dict.restype = ctypes.POINTER(ctypes.py_object)
>>> _get_dict.argtypes = [ctypes.py_object]
>>>
>>> def get_dict(obj):
...    return _get_dict(obj).contents.value
{% endhighlight %}

이제 이 get_dict를 이용해서 Patch를 해봅니다!

{% highlight python %}
>>> get_dict(date)['today'] = patch_today
{% endhighlight %}

오, 오 되었습니다. 이제 실행을 해보면!

{% highlight python %}
>>> date.today()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unbound method patch_today() must be called with date instance as first argument (got nothing instead)
{% endhighlight %}

이제는 다른 오류가 반겨줍니다. 아무래도 date의 규격에 맞지 않는 것 같습니다.<br />클래스로 다시 Patch 코드를 작성해봅시다.

{% highlight python %}
>>> class PatchDate(object):
...    @classmethod
...    def today(cls):
...        return date(2015, 1, 1)
{% endhighlight %}

다시 한번 Patch를 하고 실행을 해보면!

{% highlight python %}
>>> get_dict(date)['today'] = PatchDate.today
>>> date.today()
datetime.date(2015, 1, 1)
{% endhighlight %}

뙇!

Patch를 성공적으로 수행했으니 이만 글을 마칩니다. 읽어주셔서 감사합니다.<br />
(첫 번째 글이 너무 진지한 것 같아 이번에는 가벼운 느낌으로 글을 작성해보았습니다.)
<br />
<br />
###참고
* [http://stackoverflow.com/questions/6738987/extension-method-for-python-built-in-types](http://stackoverflow.com/questions/6738987/extension-method-for-python-built-in-types)
* [https://docs.python.org/2/library/ctypes.html](https://docs.python.org/2/library/ctypes.html)