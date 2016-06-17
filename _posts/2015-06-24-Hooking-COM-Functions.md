---
layout: post

title: Hooking COM Functions for fun and profit
cover_image: blog-cover.png

excerpt: "후킹, 어디까지 해봤니?"

author:
  name: 박경덕
  bio: Software Engineer
  page: javascript:alert('홈피 없어요...')
---


<style>
.image-wrapper {
    text-align: center;
}

.image-caption {
	display:block;
    color: #777;
    font-size:.8em;
    margin-top:-.8em;
    margin-bottom:1.2em;
}
</style>

안녕하세요. [애드투페이퍼](http://www.add2paper.com)에서 클라이언트 개발을 맡고 있는 박경덕이라고 합니다.<br>
지난번 [API 후킹 꿀팁 모음]({% post_url 2015-06-15-Complex-API-Hooking-Model %})에 이어 오늘은 COM 객체의 함수를 후킹하는 법을 알아보겠습니다.

## COM

COM 함수를 후킹하려면 먼저 COM이 뭔지를 좀 알아봐야 할 거 같습니다. COM이 뭘까요?<br>
<del>저도 잘 몰라요. 깔깔깔깔</del>

MS에서 정의하고 있는 바에 의하면 COM은 <strong>서로 다른 프로세스 혹은 머신 사이에서 어떤 객체를 쉽게 다루기 위한 기술</strong>이라고 합니다.
여기서 객체란 어떤 기능을 수행하는 클래스라고 보시면 될 것 같습니다.
COM의 상세한 스펙이나 동작 원리 등에 관해서도 물론 쓸 수는 있지만, 오늘의 주제에서 벗어난 내용이기도 하고 내용도 너무 기므로 관심 있으신 분들은 [여기](https://msdn.microsoft.com/en-us/library/windows/desktop/ms694363(v=vs.85).aspx)를 참고하세요. (몰라서 그러는 거 아닙니다. 진짜)

## COM 인터페이스

COM 객체의 실제 구현 부는 사용자의 눈에 안 보이도록 숨겨져 있습니다. 대신, 이 기능을 사용할 수 있도록 인터페이스를 두어 간접적인 접근을 허용합니다.
이러한 인터페이스는 실제 런타임에 객체의 CLSID와 IID를 이용하여 인스턴스를 생성하기 전까지는 단순한 껍데기에 불과합니다.
가령 C++에서 COM 인터페이스는 이런 식으로 선언되어 있습니다.

{% include _image.html img="/images/com_interface.png" caption="그림1 - IFileOpenDialog 인터페이스" %}

왠지 폼나게 생긴 것 같지만 결국 길바닥에 널린 한낱 구조체에 불과합니다. 안에 들어있는 함수들이 전부 순수 가상 함수로 선언된 거 보이시죠?
껍데기라니까요.

'그럼 뭐 어쩌라고?' 라고 생각하실 분들을 위해 지금부터 실제 API를 가지고 후킹을 하는 과정을 하나하나 따라가 보도록 하겠습니다.

## 우리는 한다 후킹을

Windows Vista 이상부터 파일 열기/저장 Dialog를 위한 COM 객체가 새로 추가되었으며, 이러한 Dialog를 Common Item Dialog라고 부릅니다.
오늘은 여기에 사용되는 API 중 하나를 골라 후킹 해보겠습니다.

파일을 열 때 나타나는 Dialog는 다음과 같이 생겼습니다.

{% include _image.html img="https://i-msdn.sec.s-msft.com/dynimg/IC420398.png" caption="그림2 - 크고 아름다운 Dialog" %}

오 꽤 깔쌈합니다. 이런 걸 띄우려면 <strong>IFileOpenDialog</strong> 라고 하는 인터페이스를 사용해야 합니다(그림1에 있습니다). 기왕 말이 나온 김에 어떻게 쓰는 건지나 한번 봅시다.

{% highlight C++ linenos %}
#include <iostream>
#include <atlbase.h>
#include <ShObjIdl.h>

int main()
{
    CoInitialize(0);

    CComPtr<IFileOpenDialog> dlg;
    dlg.CoCreateInstance(CLSID_FileOpenDialog, 0, CLSCTX_ALL);

    dlg->Show(0);

    CComPtr<IShellItem> item;
    if (dlg->GetResult(&item) == S_OK) // 이걸 후킹 해봅시다.
    {
        PWSTR path;
        item->GetDisplayName(SIGDN_FILESYSPATH, &path);
        std::wcout << path << std::endl;
        CoTaskMemFree(path);
    }

    CoUninitialize();

    return 0;
}
{% endhighlight %}
어때요. 참 쉽죠?

그림1에서 볼 수 있듯이 <strong>IFileOpenDialog</strong> 인터페이스는 <strong>IFileDialog</strong>를 상속하고 있고 대부분의 API는 <strong>IFileDialog</strong>에 구현되어 있습니다.
실제 사용 가능한 API는 이것저것 많지만, 지금은 별 필요 없는 것들이기 때문에 관심 있으신 분들은 [참고문헌](https://msdn.microsoft.com/en-us/library/windows/desktop/bb775966(v=vs.85).aspx)을 참고하시면 되겠습니다.

이제 위 코드에서 후킹 할 API를 찾아야 하는데 <strong>GetResult</strong> 말고는 아무리 눈을 씻고 찾아보아도 쓸 만한 게 보이질 않습니다(GetResult는 IFileDialog에 선언되어 있습니다). 이 API를 후킹 하면 특정 파일 혹은 모든 파일을 못 열게 한다든지 정해진 시간대에서는 파일을 못 열게 한다든지 하는 <del>변태적인</deL>재미난 일을 할 수가 있을 겁니다.

후킹을 하려면 후킹 할 API의 주소, 즉 구현 부의 주소를 알아야 합니다. 그런데 앞에서 COM 객체의 구현 부는 숨겨져 있다고 했단 말이죠...
컴파일러는 대체 어떻게 주소를 알아낼까요? 아무래도 저 예제 프로그램을 직접 분석해볼 수밖에 없을 것 같습니다.

{% include _image.html img="/images/com_disasm.png" caption="그림3 - WinDBG로 분석한 결과" %}

불필요한 부분은 깔끔히 쳐내고 GetResult 함수를 호출하는 부분만 잘라왔습니다. 짧은 코드지만 이 부분의 루틴을 이해해야 나중에 실제로 후킹 코드를 작성할 수 있습니다.

함수 호출 전 코드 세 줄을 보겠습니다.
{% highlight asm %}
mov ecx, dword ptr [ebp-104h]  # ebp - 104h에는 IFileOpenDialog 객체의 주소가 담겨 있습니다.
mov edx, dword ptr [ecx]       # IFileOpenDialog의 vtable을 가져옵니다.
mov eax, dword ptr [edx+50h]   # vtable + 50h에 GetResult의 주소가 있나 보군요.
{% endhighlight %}

코드 편집 과정에서 잘렸지만, ebp-104h에는 <strong>IFileOpenDialog</strong> 객체 인스턴스의 주솟값이 들어 있습니다. 코드 첫 줄에서는 먼저 이 값을 ecx 레지스터에 저장합니다.
그리고 다음 줄에서 그 주소를 다시 참조하여 edx 레지스터에 저장하는데 저게 뭐 하는 거죠?

C++에서 클래스가 가상 함수를 갖는 경우 이를 동적으로 binding 하기 위한 메커니즘이 필요합니다. 이를 위해 대부분 컴파일러는 vtable을 이용합니다.
각 가상 함수에 대해 실제 호출되어야 하는 함수의 주소를 배열로 저장해두는 것이죠.[^1] 보통 이 vtable의 주소는 인스턴스 메모리의 가장 처음에 저장됩니다.[^2]
두 번째 줄이 하는 일은 바로 이 vtable의 주소를 가져오는 겁니다.

원칙적으로 vtable은 일개 프로그래머 따위가 함부로 건드릴 만한 것이 못 됩니다. 건드려서도 안 돼요. 하지만 어쩌겠어요? 방법이 없는걸. 껄껄껄

마지막 줄에서는 가져온 vtable에서 GetResult 함수의 주소를 구하고 있습니다.
32bit 환경에서 테스트한 코드이기 때문에 주소의 크기가 4바이트라는 것을 생각하면 vtable[0x50 / 4]에 GetResult 함수 주소가 있겠군요.
역시 다른 데서 이러면 혼납니다.

좋습니다. 이제 GetResult의 주소를 구할 수 있습니다. 하지만 아직 API가 요구하는 정확한 인자(Parameter) 정보가 부족합니다. 함수를 한 번 뜯어(?)봅시다.

{% include _image.html img="/images/com_disasm2.png" caption="그림4 - GetResult 함수의 속살" %}

GetResult는 CFileOpenSave 클래스의 멤버 함수입니다. 그런데 흥미롭게도 Calling convention이 thiscall이 아니라 stdcall이네요?
일반적으로 비정적 클래스 멤버 함수의 경우 ecx 레지스터를 통해 this 객체가 전달되는 것이 규칙이지만[^3], COM 특성상 그렇게 사용할 수는 없습니다.
그래서 stdcall을 사용하고 this 객체를 스택을 통해 넘깁니다(그림3의 0x1bb1e6번지와 0x1bb1ed번지 두 곳에서 인자를 push 하는 것을 확인해보세요).

## 후킹 코드 작성

이제 모든 준비가 됐습니다. 후킹 할 함수의 주소도 알아낼 수 있고, 그 함수의 실제 Prototype도 확인했습니다.
이전 포스트에서 배운 꿀팁을 이용하여 간단하게 후킹 코드를 작성해봅시다.

{% highlight C++ linenos %}
// 함수 선언부
typedef HRESULT(STDMETHODCALLTYPE IFileDialog::*GetResultFunction)(IShellItem **);
Hook::OverrideHook<GetResultFunction> GetResultHook;

// 생략...

void hook()
{
    CComPtr<IFileOpenDialog> dlg;
    dlg.CoCreateInstance(CLSID_FileOpenDialog, 0, CLSCTX_ALL);

    size_t *pvtable = (size_t *)dlg.p;     // vtable의 주소를 구해옵니다.
    if (!pvtable) return;

    size_t *vtable = (size_t *)pvtable[0]; // 실제 vtable을 구해옵니다.

    const int idx = 0x50 / 4;     // 32bit 환경에서 테스트했기 때문에 4로 나눠줍니다.

    GetResultHook.Install(vtable[idx],
    [](IFileDialog *_this, IShellItem **ppsi)
    {
        // 먼저 원본 함수를 호출하여 결과를 얻어옵니다.
        auto ret = (_this->*GetResultHook)(ppsi);

        // 이제 여기서 재미난 작업들을 합니다.

        // 결과를 리턴합니다(재미난 작업에 따라 ret 값이 바뀔 수도 있겠네요).
        return ret;
    });
}
{% endhighlight %}

별다른 설명이 필요 없을 정도 심플하고 깔끔합니다. 절로 미소가 지어지네요(<del>사실 테스트는 안 해ㅂ...</del>).

그렇다고 진짜 아무 설명도 없이 넘어가면 모양새가 좀 그러니 굳이 하나를 집어보자면 23번째 라인에 원본 함수를 호출하는 부분의 문법이 좀 특이하다는 걸 볼 수 있습니다.
-> 다음에 *가 붙다니... 아무리 봐도 마음에 들지 않는 모양새입니다. 그런데 이게 하나의 연산자랍니다. 정말이에요. 아직도 못 믿겠다 하시는 분들은 [이거](https://msdn.microsoft.com/en-us/library/k8336763.aspx) 한 번 보세요. 진짜죠?

## 마치며

COM 함수를 후킹 하는 것은 사실 저도 이번에 클라이언트를 개발하면서 처음 해봤습니다.
이 글에는 그냥 이렇게 저렇게 하면 된다고 굉장히 간단하게 썼지만, 실제 처음 개발할 때는 이런저런 시행착오를 많이 겪었습니다.
API 함수의 주소를 구하는 법도 많이 고민하고 Calling convention을 맞추느라 삽질도 많이 했죠...
그래도 결국은 어떻게든 방법을 찾아서 기능도 구현하고 이렇게 글도 쓰고 하니 기쁘네요 :)

본문에서도 언급했지만 제가 후킹 코드에 쓴 것처럼 vtable이나 가상 함수 오프셋 등을 직접 가지고 노는 것은 절대 추천할 만한 게 못 됩니다.
저런 애들이랑 놀다간 언제 어떻게 멘탈이 터질지 알 수 없단 말입니다.
저는 다른 방법이 도저히 떠오르지 않아서 어쩔 수 없이 사용했지만, 혹시 이 글을 읽으신 여러분들 중 더 좋은 방법을 찾으신 분이 계신다면 당장 저놈들을 내쫓으셔야 합니다.<br>
물론 저한테도 좀 알려주세요!

그럼 이쯤에서 이만 줄이고 다음번에 더 흥미로운 주제로 찾아뵙도록 하겠습니다.

감사합니다.

[^1]: 사실 C++ 표준에 의하면 가상 함수 binding은 런타임에 하게 되어 있지만, 성능상의 이유로 대부분 이를 컴파일 타임에 처리합니다.
[^2]: 그렇다고 vtable이 항상 오프셋 0에 있을 거라고 장담할 순 없습니다. 컴파일러 제작사마다 vtable을 다른 곳에 저장할 수도 있고, 다중 상속이나 가상 상속이 쓰이는 경우 오프셋이 달라질 수도 있습니다.
[^3]: 32bit Visual C++ 환경일 경우에만 해당합니다.

---