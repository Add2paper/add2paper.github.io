---
layout: post

title: API Hooking Honey-Butter Tips
cover_image: blog-cover.png

excerpt: "API 후킹 꿀팁 모음"

author:
  name: 오지헌
  bio: Software Engineer
---

[애드투페이퍼](http://www.add2paper.com)의 시스템 개발자 오지헌입니다. API 후킹을 통해 조금 복잡한 애플리케이션을 개발하고 싶은 분들을 위해 몇 가지 팁을 모아봤습니다.

## 후킹 방법 선택하기
### 커널레벨
	
루트킷이나 백신 등이 사용하는 가장 높은 권한의 후킹 방식입니다.
개발 난이도는 높습니다.
요즘은 코드사인이 되지 않은 드라이버는 메모리에 로딩하는 것 조차 불가능하기 때문에 여러 의미로 개인 개발자들이 사용하기 힘든 후킹 방법입니다.

	드라이버 개발, HW/OS 지식이 필요
	사소한 실수에 블루스크린을 볼 수 있음
	OS별, CPU 아키텍처별 별도 개발이 필요
	코드사이닝 인증서 필요
	
### 유저레벨
	
주로 DLL을 작성하여 프로세스 단위에 적용하는 안정성, 호환성이 높은 후킹 방식입니다. 추천.
	
* IAT 후킹

	대표적 유저레벨 후킹 방식의 하나로, 모듈 별 후킹 여부를 결정할 수 있습니다. API의 주소가 담긴 테이블을 수정하는 식으로 작동합니다.
	
* Trampoline

	API의 코드를 직접 패치하는 방식입니다. 때문에 후킹이 한 프로세스 내에서 전역적으로 적용됩니다. 제대로 구현하기가 IAT 후킹에 비해 훨씬 까다롭습니다.
	
## 글로벌 후킹

유저레벨 API 후킹을 하게 된다면 한 가지 문제가 생깁니다. <br />
윈도우에서 모든 프로세스는 각각 독립적인 가상메모리 공간을 가지고 있으므로, API 후킹은 한 프로세스에 대해서만 적용됩니다. <br />
때문에 글로벌 후킹을 위해선 모든 프로세스에 각각 후킹을 걸어주어야 합니다.

가장 간단한 방법으로 [SetWindowsHookEx](https://msdn.microsoft.com/en-us/library/windows/desktop/ms644990.aspx) API를 사용하는 방식을 추천합니다. <br/>
후킹을 거는 Dll을 작성하고, 위 API로 Dll이 '윈도우를 사용하는 모든 프로세스'에 로드되도록 명령하면 됩니다.

## 허니버터꿀팁

API 후킹의 개발 방법론이나 팁은 구글에서도 쉬이 찾을 수 없기에, 간략히 정리해봅니다. <br/>
우선 다른 어플리케이션이 특정 프로세스(이름이 abc로 시작하는 프로세스)를 종료하지 못하게 막는 상황을 가정해보겠습니다. <br />

### 소유권 관리
API 후킹에 핸들/객체 소유권 관리의 개념이 도입되면 간단한 구현으로 매우 큰 개발 생산성 향상을 취할 수 있습니다.

우선 종료 요청을 수정해야하기 때문에 TerminateProcess의 후킹은 필수적입니다.
TerminateProcess의 원형은 아래와 같습니다.

		BOOL WINAPI TerminateProcess(
		  _In_ HANDLE hProcess,
		  _In_ UINT   uExitCode
		);

소유권 관리 개념이 없다면, 아마 후킹 프로시저는 아래과 같아질 겁니다(슈도코드).
		
		HookedTerminateProcess(process, exitCode)
		{
			name = getProcessName(process)
			if checkAllowedProcessName(name) then 호출허용
			else 호출거부
		}

척 보기에도 문제점이 많습니다. <br/>
getProcessName, checkAllowedProcessName는 매번 호출되기에는 무거워보입니다. 뿐만 아니라 주어진 핸들의 권한이 부족해 getProcessName 함수가 실패할 수도 있습니다!

소유권 관리 개념이 추가되면, 코드는 아래와 같아집니다.
		
		Process::Process(handle, pid, access, ...)
		{
			allowed = checkAllowedProcess(...);
			handle_ = handle;
			pid_ = pid;
		}
		Process::~Process()
		{ handle 등 정리 }

		HookedOpenProcess(pid, access, ...)
		{
			opened_handle = OriginalOpenProcess(...)
			if opened_handle != null then processes.add(shared_ptr<Process>(new Process(opened_handle, pid, access, ...)));
			return opened_handle
		}
		HookedCloseHandle(handle)
		{
			if processes.exists(handle) then processes.remove(handle)
			else return OriginalCloseHandle(...)
		}
		HookedTerminateProcess(process, exitCode)
		{
			myProcess = processes.find(process)
			if myProcess->IsAllowed() then 호출허용
			else 호출거부
		}

* OpenProcess와 CloseHandle이 추가적으로 후킹됐습니다. <br/>
* OpenProcess에선 내부적으로 Process객체를 만들어 프로세스 핸들의 소유권을 넘겨주며<br/>
    CloseHandle에선 Process객체의 reference counter를 하나 줄입니다. <br/>
1. checkAllowedProcess()는 핸들 별로 단 한번만 호출되어 성능이 향상됩니다. <br/>
2. 핸들뿐 아니라 pid가 주어졌기 때문에 원하는 권한으로 프로세스 이름을 쿼리할 수 있습니다. <br/>
3. 핸들을 가지고 해당 핸들의 pid나 access rights를 알아낼 수 있습니다. <br/>
4. 또한, CloseHandle이 호출되더라도 핸들을 즉시 닫지 않고 원하는 만큼 사용하다 닫을 수 있습니다. <br/>

개이득.

###	SetLastError

후킹 프로시저 내에서 요청을 거부했을 경우, 단순히 FALSE를 리턴하지 않고 SetLastError()를 사용해 실패 이유를 알려주면 오동작을 방지할 수 있습니다. <br/>
가령, TerminateProcess 요청을 실패시킬 때

		SetLastError(ERROR_ACCESS_DENIED)
		return FALSE

로 오류코드를 ERROR_ACCESS_DENIED로 명시적으로 설정해주면 잘 만든 프로그램들은 스스로 GetLastError()로 오류코드를 체크해 그에 맞는 메세지를 출력해주거나 별도의 작업을 수행합니다.

### Meta programming

Variadic Template을 사용하면

		Hook::OverrideHook<decltype(&MessageBoxA)> messageBoxAHook // 선언

		messageBoxAHook.Install(
			api주소,
			[this](...) // 후킹프로시저
		{
			return messageBoxAHook(hWnd, "[hooked] " + lpText, ...)
		})
	
이렇게 편하고 type-safe한 API 후킹을 할 수 있다. <br/>
물론 함수 prototype만 보고 hooking stub을 만들어야 하기 때문에 머리가 빠개집니다.

### 마치며
쓰다보니 점점 귀찮아져서 내용이 부실해지네요. 나머진 다음 글에서!

<br />
### 유용한 글
* [http://www.reversecore.com/54](http://www.reversecore.com/54)
* [http://www.codeproject.com/Articles/2082/API-hooking-revealed](http://www.codeproject.com/Articles/2082/API-hooking-revealed)