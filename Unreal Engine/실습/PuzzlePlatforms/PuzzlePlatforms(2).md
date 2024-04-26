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

### BindWidget을 통한 Widget C++연결하기

Widget에 하위 위젯을 C++에서 연결하는 방법이 있는데 UPROPERTY(meta = (BindWidget))이다.

이를 통해서 같은 타입의 클래스가 Widget에 하위 속성으로 있고 그 이름이 변수명과 같다면 해당 Widget변수에 바인딩 된다.

```C++
UCLASS()
class PUZZLEPLATFORMS_API UMainMenu : public UUserWidget
{
	GENERATED_BODY()

public:
	void SetMenuInterface(IMenuInterface *InstanceMenuInterface);

protected:
	virtual bool Initialize();

private:
	// BindWidget : 해당 서브 위젯과 변수의 이름을 바탕으로 자동으로 매칭해서 바인딩 해주는 옵션(따라서 Button 서브 위젯의 이름도 Host여야 같은 UButton Host변수에 매칭된다.)
	UPROPERTY(meta = (BindWidget))
	class UButton *Host;

	UPROPERTY(meta = (BindWidget))
	class UButton *Join;

	UFUNCTION()
	void HostServer();

	UFUNCTION()
	void JoinServer();

	IMenuInterface *MenuInterface;
};
```

이를 통해서 Host와 Join에 메뉴 블루프린트에서 만든 버튼들이 매칭됐기에 이를 델리게이트에 등록해서 콜백 함수로 만들어준다.

이 때 사용된 메소드는 OnClicked.AddDynamic()이다.

```C++
// Fill out your copyright notice in the Description page of Project Settings.

#include "MainMenu.h"
#include "Components/Button.h"

bool UMainMenu::Initialize()
{
    bool Success = Super::Initialize();

    if (!Success)
        return false;

    // TODO: Setup
    if (Host == nullptr || Join == nullptr)
        return false;

    Host->OnClicked.AddDynamic(this, &UMainMenu::HostServer);
    Join->OnClicked.AddDynamic(this, &UMainMenu::JoinServer);

    return true;
}
```

### Interface를 통한 의존성 주입

만들어놓은 메뉴는 게임 인스턴스와는 독립된 시스템으로 존재하게 하는 것이 생산성이나 재활용성에서 훨씬 효율적이다. 그렇기에 메뉴 시스템이 게임에 종속되지 않도록 하는 것이 이상적인 구조인데 이를 위해서는 둘을 각각의 클래스로 만들고 의존성 주입을 통해서 동작이 되도록 하는 구조로 만들고자 한다.

MenuWidget에서는 여러 게임 Instance에서 사용가능하게 하기 위해서 의존성 주입을 하는 방법으로 Interface를 사용하게 된다.

Interface에서 Widget이 사용하고자 하는 메소드들을 순수 가상 함수로 선언하고 이 Interface를 상속받은 GameInstance에서 의존성 주입을 Widget에게 의존성을 주입해주면 서로 독립적인 시스템을 만들 수 있다. 이를 코드로 보면 아래와 같다.

-MainMenu-<br>
Main Menu에서는 아래와 같이 Widget에서 사용될 다른 클래스의 기능을 Interface의 메소드를 통해서만 동작시키게 된다.

```C++
// MainMenu.h
#include "MenuInterface.h"
#include "MainMenu.generated.h"

UCLASS()
class PUZZLEPLATFORMS_API UMainMenu : public UUserWidget
{
	GENERATED_BODY()

public:
	void SetMenuInterface(IMenuInterface *InstanceMenuInterface);

private:
	IMenuInterface *MenuInterface;
};


// MainMenu.cpp
void UMainMenu::SetMenuInterface(IMenuInterface *InstanceMenuInterface)
{
    this->MenuInterface = InstanceMenuInterface;
}

void UMainMenu::HostServer()
{
    UE_LOG(LogTemp, Warning, TEXT("Hosting Now!!"));
    if (MenuInterface != nullptr)
    {
        MenuInterface->Host();
    }
}
```

-MenuInterface-<br>
메뉴 인터페이스에서는 아래처럼 사용하고자하는 메소드를 순수 가상 함수로써 선언만 해놓는다.

```C++
#pragma once

#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "MenuInterface.generated.h"

// This class does not need to be modified.
UINTERFACE(MinimalAPI)
class UMenuInterface : public UInterface
{
	GENERATED_BODY()
};

/**
 *
 */
class PUZZLEPLATFORMS_API IMenuInterface
{
	GENERATED_BODY()

	// Add interface functions to this class. This is the class that will be inherited to implement this interface.
public:
	// 구현은 상속받은 곳에서 하면 된다는 뜻 =0
	virtual void Host() = 0;
};
```

-PuzzlePlatformsGameInstance-<br>
기존에 만들어뒀던 게임 인스턴스에 상속 부분을 보면 public IMenuInterface를 상속받은 것을 볼 수 있는데 그곳에 있던 Host를 Override하여서 현재 Host의 기능이 Menu에서 Interface를 통해서 동작할 수 있게 됐다.

또 중요한 것으로는 cpp에서 종속성 주입을 하여서 Menu에서 현재 이 게임 인스턴스를 Menu에서 조작할 instance임을 알려주는 의존성 반전을 구현하는 것임을 이해해야 한다.

```C++
UCLASS()
class PUZZLEPLATFORMS_API UPuzzlePlatformsGameInstance : public UGameInstance, public IMenuInterface
{
	GENERATED_BODY()

public:
	UFUNCTION(Exec)
	void Host();

};

// CPP

void UPuzzlePlatformsGameInstance::LoadMenu()
{
    if (MenuClass == nullptr)
        return;
    UMainMenu *Menu = CreateWidget<UMainMenu>(this, MenuClass);

    if (Menu == nullptr)
        return;

    Menu->AddToViewport();

    // 종속성 주입
    Menu->SetMenuInterface(this);
}
```

이를 통해서 인터페이스를 통한 의존성 반전과 종속성 주입을 통해서 각 클래스를 독립적으로 운용할 수 있게 되는 점을 이해하는 것이 중요하다.

### MenuSwitcher를 통한 Widget변경

이전에 사용했던 binding기능을 사용해서 MenuSwitcher를 가져온 뒤 C++ 코드에서 버튼을 클릭시 원하는 위젯으로 변경하는 코드를 구성했다.

다른 부분은 빼고 MenuSwicher에서 SetActiveWidget()을 통해서 다른 위젯으로 변경하는 코드는 아래와 같다.

```C++
void UMainMenu::OpenJoinMenu()
{
    if (JoinMenu == nullptr)
        return;

    MenuSwitcher->SetActiveWidget(JoinMenu);
}
```

### TextField에 입력값으로 서버에 Join하기

TextField에서 GetText()를 통해서 해당 값을 가져와서 접속을 시도한다. 전체적인 다른 구조는 이전에 Host()기능 구현과 비슷하기에 이 부분에 대해서는 생략한다.

```C++
void UMainMenu::JoinServer()
{
    if (MenuInterface != nullptr)
    {
        if (IPAddressField == nullptr)
            return;

        const FString &Address = IPAddressField->GetText().ToString();
        MenuInterface->Join(Address);
    }
}
```

### 서버에서 나가기 기능

인게임에서 특정 플레이어가 종료하고 나가는 경우에는 해당 플레이어 컨트롤러만 이동해야하기에 ClientTravle를 사용해서 구현헀다.

InGameMenu 에서 quit 버튼을 눌러서 게임 종료 지시

```C++
void UInGameMenu::QuitPressed()
{
    if (MenuInterface != nullptr)
    {
        Teardown();
        MenuInterface->LoadMainMenu();
    }
}
```

게임 인스턴스에서는 해당 플레이어 컨트롤러를 찾아 ClientTravel로 MainMenu로 이동

```C++
void UPuzzlePlatformsGameInstance::LoadMainMenu()
{

    APlayerController *PlayerController = GetFirstLocalPlayerController();

    if (PlayerController == nullptr)
        return;

    PlayerController->ClientTravel("/Game/MenuSystem/MainMenu", ETravelType::TRAVEL_Absolute);
}

```

### 게임 종료

게임 클라이언트 자체를 종료시키는 방법은 여러가지가 있으나 이번에 사용한 방식은 플레이어 컨트롤러에서 ConsoleCommand()에 quit 명령어를 입력하는 기능을 나가기 버튼에 넣어서 구현하는 방식이었다.

종료 버튼을 누르면 플레이어 컨트롤러에서 quit 명령어를 입력하는 방식이다.

```C++

void UMainMenu::QuitPressed()
{

    UWorld *World = GetWorld();
    if (World == nullptr)
        return;
    APlayerController *PlayerController = World->GetFirstPlayerController();
    if (PlayerController == nullptr)
        return;

    PlayerController->ConsoleCommand("quit");
}
```
