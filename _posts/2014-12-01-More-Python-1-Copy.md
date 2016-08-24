---
layout: post

title: More Python 1 - Copy
cover_image: blog-cover.png

excerpt: "Python의 Copy 관련 내용을 풀어보겠습니다!"

author:
  name: 박아윤
  bio: Software Engineer
  image: pay.jpg
  page: http://www.parkayun.kr
---

안녕하세요, [애드투페이퍼](http://www.add2paper.com)에서 백엔드를 개발하고 있는 [박아윤](http://www.parkayun.kr)입니다.<br />
이번 포스트부터 Python의 멀고도 가까운 내용을 소개하는 More Python을 연재코자 합니다. <br />
그 첫 번째 포스트로 Python의 Copy 관련 내용을 풀어보겠습니다!

<br />

## Copy
이 포스트에서 말하고자 하는 Copy는 변수를 복사하는 그 Copy입니다. <br />예를 들어 아래와 같이 apple 변수 값을  banana 변수에 그대로 대입하는 경우입니다.
{% highlight python %}
>>> apple = 'red'
>>> banana = apple
{% endhighlight %}

<br />

그럼 그냥 쓰면 되지 문제가 뭐길래 포스팅까지 하느냐? 하면 바로 아래와 같은 경우입니다.
{% highlight python %}
>>> foo = [0, 1, 2]
>>> bar = foo
>>> foo[0] = 9
>>> print bar
[9, 1, 2]
{% endhighlight %}

<br />

foo를 초기화하고 bar에 값을 대입하고 foo의 값을 바꿨더니 bar의 값도 따라 바뀌었습니다.<br />
아래에서 확인할 수 있듯이 값을 대입하면 값에 대한 메모리가 더 할당되는 것이 아닌 기존 값의 메모리 주소를 공유하기 때문에 발생하게 됩니다. 리스트 같은 경우 리스트 자체뿐만 아니라 리스트 내 요소들도 똑같은 주소를 공유합니다. 이러한 Copy를 **Shallow Copy(얕은 복사)**라고 합니다.
{% highlight python %}
>>> foo = [0, 1, 2]
>>> bar = foo
>>> print id(foo), id(bar), id(foo) == id(bar)
4433166064, 4433166064, True
>>> print id(foo[0]), id(bar[0]), id(foo[0]) == id(bar[0])
140327877880784, 140327877880784, True
>>> foo[0] = 777
>>> print id(foo[0]), id(bar[0]), id(foo[0]) == id(bar[0])
140327876950296, 140327876950296, True
{% endhighlight %}

<br />

그럼 이러한 현상을 피하기 위해서는 무엇이 필요할까요? 얕은 복사의 반의어인 **Deep Copy(깊은 복사)**를 이용하면 됩니다. Python에서는 기본적으로 copy 모듈을 제공하고 있어 아래와 같이 사용하면 됩니다.
{% highlight python %}
>>> import copy
>>> foo = [0, 1, 2]
>>> bar = copy.deepcopy(foo)
>>> foo[0] = 123
>>> print id(foo), id(bar), id(foo) == id(bar),
4433166064, 4433319624, False
>>> print foo, bar
[123, 1, 2] [0, 1, 2]
{% endhighlight %}

<br />

이렇게 Deep Copy를 하더라도 리스트 내 요소들은 여전히 똑같은 값을 공유하고 있습니다. 그러나 얕은 복사와는 다르게 리스트 자체는 다른 값이기 때문에 원본 리스트의 요소가 바뀌었다고 복사된 리스트의 요소에 영향을 끼치지는 않습니다.
{% highlight python %}
>>> import copy
>>> foo = [0, 1, 2]
>>> bar = copy.deepcopy(foo)
>>> print id(foo[0]), id(bar[0]), id(foo[0]) == id(bar[0])
140327876950296, 140327876950296, True
>>> foo[0] = 777
>>> print id(foo[0]), id(bar[0]), id(foo[0]) == id(bar[0])
140327877880712, 140327876950296, False
{% endhighlight %}

<br />

Copy 모듈을 쓰지 않고 아래와 같은 방법으로도 Deep Copy와 비슷한 효과를 얻을 수 있습니다. 그러나 여기서 주의할 점은 효과만 비슷할 뿐 Deep Copy가 아닌 Shallow Copy입니다. 
{% highlight python %}
>>> foo = [0, 1, 2]
>>> bar = foo[:]
>>> foo[0] = 5
>>> print foo, bar
[5, 1, 2] [0, 1, 2]
{% endhighlight %}

<br />

좀 더 구체적으로 사용해보면 바로 Shallow Copy인 이유를 바로 알 수 있습니다.
{% highlight python %}
>>> foo = [[0, 1, 2]]
>>> bar = foo[:]
>>> foo[0].append(3)
>>> print foo, bar
[[0, 1, 2, 3]] [[0, 1, 2, 3]]
{% endhighlight %}

<br />

## And more?
기본적으로 리스트형뿐만 아니라 모든 변수형에 적용되는 사항이나 문자열은 값을 변경하려면 새로 대입해야 하고 튜플은 값을 변경하는 것이 목적도 아니고 바뀌지도 않으니 실질적으로 사용할 수 있는 건 리스트와 사전형입니다. 특히 사전형보다는 리스트를 다룰 때 자주 사용될 수 있으니 실제 코딩하실 때 참고하시면 도움되실듯합니다. 

다른 흥미로운 주제를 가지고 다시 찾아뵙도록 하겠습니다. :D

읽어주셔서 감사합니다.
