# FPS 프로젝트

해당 프로젝트는 언리얼에서 공식적으로 지원되는 튜토리얼을 기반으로 진행되었습니다.

[언리얼 FPS 프로젝트 튜토리얼](https://docs.unrealengine.com/4.26/ko/ProgrammingAndScripting/ProgrammingWithCPP/CPPTutorials/FirstPersonShooter/)

4단계로 나눠진 단계를 진행하게 되는데 차례대로 따라가면서 진행해보겠습니다.

## 1. 프로젝트 구성

처음 해야할 일은 프로젝트를 생성하고 게임의 기본적인 세팅을 맞춰주는 일입니다.<br>
기초적인 내용은 튜토리얼을 따라가면 되고 크게 어렵지 않기에 차근차근 따라가면서 작성하면 됩니다.

진행하게 되면 FPSProjectGameBaseMode.h에서 StarPlay()를 선언하고 FPSProjectGameBaseMode.cpp 에서 StarPlay()를 작성하게 됩니다.
이 때

```C++
void AFPSProjectGameMode::StartPlay()
{
    Super::StartPlay();

    if (GEngine)
    {
        // 디버그 메시지를 5 초간 표시합니다.
        // "키" (첫 번째 인수) 값을 -1 로 하면 이 메시지를 절대 업데이트하거나 새로고칠 필요가 없음을 나타냅니다.
        GEngine->AddOnScreenDebugMessage(-1, 5.0f, FColor::Yellow, TEXT("Hello World, this is FPSGameMode!"));
    }
}
```

해당 내용을 작성해주어서 게임 화면에서 로그를 출력해주게 합니다.

이제 작성된 파일을 컴파일 해주어 오류가 없는지 확인하고 에디터에 반영시켜야 합니다. 언리얼 4에서는 컴파일을 visual studio에서 진행하던지 에디터에서 진행하면 됐지만 5에서는 조금 다릅니다.<br>
5 에서는 에디터에 콘솔 입력창에 liveCoding.Compile 을 입력하여 보다 즉각적인 컴파일과 확인이 가능합니다.
![Alt text](/Assets/Images/Unreal/실습/FPSProject/1/6.png)

컴파일이 되었다고 해당 모드가 플레이에 반영된 것이 아닙니다. 해당 내용을 진짜 게임의 베이스 모드로 쓰겠다고 프로젝트 세팅을 해주어야 합니다. 먼저 작성된 FPSProjectGameBaseMode를 바탕으로 블루프린트로 만든 뒤 이것을 <br>
편집->프로젝트 세팅->맵 & 모드<br>
로 들어가서 아래 이미지와 같이 등록해줍니다.
![Alt text](/Assets/Images/Unreal/실습/FPSProject/1/7.png)

그런 다음에 플레이를 누르게 되면?
![Alt text](/Assets/Images/Unreal/실습/FPSProject/1/5.png)
이미지의 왼쪽 위처럼 제대로 로그가 처음에 찍히게 됩니다. 이는 즉 GameBaseMode가 제대로 변경되었음을 뜻합니다.

다음은 캐릭터에 기본적인 구현과 동작을 처리해보도록 하겠습니다.
