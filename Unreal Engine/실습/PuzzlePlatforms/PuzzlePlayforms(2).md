# PuzzlePlatforms(2)

## Menu System UI 구현

먼저 위젯 블루프린트로 MainMenu의 대략의 틀을 구현했다.

![10](/Assets/Images/Unreal/실습/PuzzlePlatforms/10.png)

### Unreal C++ 에서 UI에 대한 접근

기본적으로 C++에서 언리얼에서 지원하는 UI 기능인 UMG(언리얼 모션 그래픽)에 대한 정보가 없기에 이를 PuzzlePlatforms.Build.cs 에서 모듈을 추가시켜주어야한다. 그래서 AddRange에 UMG를 추가해주었다.

PuzzlePlatforms.Build.cs

```C++
using UnrealBuildTool;

public class PuzzlePlatforms : ModuleRules
{
	public PuzzlePlatforms(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

		PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "HeadMountedDisplay", "EnhancedInput","UMG" });
	}
}
```

### FClassFinder

블루프린트 클래스를 찾는 방법 중 하나로 FClassFinder가 있는데 이를 활용하면 원하는 블루프린트를 레퍼런스 경로를 통해서 얻어올 수 있다.

이를 통해서 만들어 놓은 MainMenu 블루 프린트의 정보를 가져와서 C++에서의 클래스화 시켰다.

```C++
//FClassFinder를 사용하기 위한 헤더 파일
#include "UObject/ConstructorHelpers.h"

UPuzzlePlatformsGameInstance::UPuzzlePlatformsGameInstance(const FObjectInitializer &ObjectInitializer)
{
    //레퍼런스를 통한 접근
    ConstructorHelpers::FClassFinder<UUserWidget> MenuBPClass(TEXT("/Game/MenuSystem/WBP_MainMenu"));
}
```

이를 통해서 얻어낸 MenuBPClass의 이름을 확인해보면 제대로 찾아냈다는 것을 알아낼 수 있다.
![11](/Assets/Images/Unreal/실습/PuzzlePlatforms/11.png)

### UI에 대한 인풋 활성화

UI 위젯을 화면에 소환하더라도 마우스 커서가 없고 위젯에 입력을 할 수 도 없는 상태로 존재한다 이를 해결하기 위해서는 플레이어 컨트롤러의 인풋 모드에서 해당 위젯에 대한 인풋 모드를 활성화 시켜주어야한다.

이를 위한 코드는 아래와 같다.

코드를 보면 UI 위젯을 소환하고 해당 위젯에 대한 FInputModeUIOnly를 만들어 필요한 옵션을 세팅해주고 PlayerController에서 SetInputMode에 만든 인풋 모드를 설정함으로써 UI를 조작할 수 있게 해준다. 또 PlayerController에게 있는 bShowMouseCursor를 통해서 마우스를 보이도록하여 조작할 수 있게 한다.

```C++
void UPuzzlePlatformsGameInstance::LoadMenu()
{
    if (MenuClass == nullptr)
        return;
    UUserWidget *Menu = CreateWidget<UUserWidget>(this, MenuClass);

    if (Menu == nullptr)
        return;

    Menu->AddToViewport();

    APlayerController *PlayerController = GetFirstLocalPlayerController();
    if (PlayerController == nullptr)
        return;

    // UI 입력 가능하게 설정하는 기능

    // UI인풋에 관한 모드를 생성함
    FInputModeUIOnly InputModeData;
    // 조작하고자 하는 위젯의 포커스를 둠
    InputModeData.SetWidgetToFocus(Menu->TakeWidget());
    // 마우스가 화면에 잠기지 않게 설정(화면 밖으로 이동하거나 하는 것이 가능)
    InputModeData.SetLockMouseToViewportBehavior(EMouseLockMode::DoNotLock);

    // 만든 인풋 모드로 컨트롤러의 인풋 모드를 변경
    PlayerController->SetInputMode(InputModeData);

    // 커서 보이기
    PlayerController->bShowMouseCursor = true;
}
```
