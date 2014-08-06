---
layout: post

title: 테스트와 CI
cover_image: blog-cover.png

excerpt: "테스트와 CI에 대하여 소개합니다."

author:
  name: 박아윤
  bio: Software Engineer
  image: pay.jpg
  page: http://www.parkayun.kr
---

안녕하세요, [애드투페이퍼](http://www.add2paper.com)에서 백엔드 개발을 맡은 [박아윤](http://www.parkayun.kr)입니다. 이번 블로그 개편에 맞춰 저희가 하고 있는 백엔드 테스트, 장고 테스트에 대하여 공유하고자 합니다.

##Overview
현재 애드투페이퍼 백엔드는 API와 웹 서비스로 구성되어있습니다. 이 중에서 API 서비스 테스트에 관하여 소개하고자 합니다.

##Test?
테스트는 흔히 통상적인 개념으로 무엇인가를 검사하는 행위를 지칭합니다.  
[소프트웨어 테스트](http://ko.wikipedia.org/wiki/%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4_%ED%85%8C%EC%8A%A4%ED%8A%B8)도 이와 마찬가지로 소프트웨어가 올바르게 의도한 대로 작동하는가를 검사하는 것 이라고 할 수 있습니다. 이러한 소프트웨어 테스트는 버그 발견뿐만 아니라 테스트와 [리팩토링](http://ko.wikipedia.org/wiki/%EB%A6%AC%ED%8C%A9%ED%86%A0%EB%A7%81) 과정을 통해서 코드 품질 향상에도 좋은 영향을 이끌어옵니다.  
이런 연유와 유사하게 소프트웨어 개발 기법으로 [테스트 주도 개발](http://ko.wikipedia.org/wiki/%ED%85%8C%EC%8A%A4%ED%8A%B8_%EC%A3%BC%EB%8F%84_%EA%B0%9C%EB%B0%9C)이라는 기법도 탄생하게 됩니다.

##Django Test
장고 프레임워크에서는 기본적으로 [장고 테스트 모듈](https://docs.djangoproject.com/en/1.7/topics/testing/overview/)을 제공합니다.  
장고 테스트 모듈에는 [파이썬 유닛 테스트](https://docs.python.org/2/library/unittest.html)와 유사하나 가장 큰 이점은 자동으로 테스트 데이터베이스를 생성해주고 끝나면 삭제까지 해줍니다.

    from django.http import HttpResponse
    
    
    def HelloView(request):
        return HttpResponse('Hello World')

이 HelloView의 기본적인 테스트 조건은 아래와 같습니다.
* Response를 제대로 해주는가
* 의도한 결괏값인가  

위 조건대로 장고 테스트 모듈을 이용하여 테스트 코드를 작성해 보겠습니다.

    from django.test import TestCase
    
    
    class HelloTestCase(TestCase):
        #테스트전 초기화하는 함수입니다.
        def setUp(self):
            self.baseURL = '/'
            
        def test_hello(self):
            url = self.baseURL + 'hello/'
            
            #성공 테스트
            res = self.client.get(url)
            self.assertEqual(res.status_code, 200)
            self.assertEqual(res.content, 'Hello')
              
위 코드와 같은 형태로 테스트 코드를 작성할 수 있습니다. 이때 유의할 점은 test\_hello와 같이 테스트 함수는 함수명 앞에 test\_와 같이 네이밍을 해줘야 합니다. 또한, 이 코드는 간단한 예시이지만 실제로 코드를 작성할 때는 성공하는 테스트뿐만 아니라 실패하는 테스트 코드도 작성해줘야 합니다. 예를 들어 뷰에서 post 요청만을 받는다면 get으로 요청 시에 정상적으로 거절하는지, 실패하는지에 대해서도 테스트를 해야 합니다.

##CI
프로젝트 규모가 커지면 빌드와 테스트하는 데에 많은 자원적, 시간적 비용이 들게 됩니다. 이런 문제점을 해결하기 위해 한 시스템에서 빌드와 테스트를 수행하는데 이를 [CI(Continuous Integration)](http://ko.wikipedia.org/wiki/%EC%A7%80%EC%86%8D%EC%A0%81%EC%9D%B8_%ED%86%B5%ED%95%A9) 시스템이라고 합니다.  
현재 애드투페이퍼에서는 [codeship.io](https://codeship.io)이라는 서비스에 소스 코드 저장소의 훅을 걸어 사용하고 있습니다. 코드를 수정하고 저장소에 푸쉬를 하게 되면 아래와 같은 흐름대로 코드 테스트를 진행합니다.

* 환경변수 설정
* 저장소 클론
* 저장소 업데이트
* 코드 폴더로 이동
* 의존성 캐쉬 설정
* 가상 머신 설정
* 의존성 설치
* 데이터베이스 싱크
* 테스트 수행
* 결과 전달  

![코드쉽 테스트](/images/codeship_test.png)  

위 스크린 샷처럼 푸쉬를 할 때마다 테스트를 수행하고 아래 스크린 샷처럼 그 결과를 메일이나 힙챗과 같은 수단으로 테스트 성공 여부를 받을 수 있습니다. 그리고 테스트가 성공하면 릴리즈 서버로 배포할 수 있도록 설정할 수도 있습니다.

![테스트 피드백](/images/test_feedback.png)  

##And more?
이 밖에도 백엔드에 관하여 공유사항들이 많이 있는데요, 다음 포스팅에서 공유하도록 하겠습니다.

읽어주셔서 감사합니다.
