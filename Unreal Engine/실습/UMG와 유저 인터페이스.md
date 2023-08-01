# UMG와 유저 인터페이스

이번에 실습할 내용은 언리얼에서 지원하는 인터페이스 제작 툴과 이를 이용한 간단한 인터페이스를 제작해보는 내용입니다.

[실습 링크 : UMG와 유저 인터페이스](https://docs.unrealengine.com/4.26/ko/ProgrammingAndScripting/ProgrammingWithCPP/CPPTutorials/UMG/)

해당 내용을 따라가면서 진행해보겠습니다.

## UMG

먼저 UMG가 무엇인지 궁금해서 찾아보았습니다.<br>

UMG(Unreal Motion Graphic)는 언리얼에서 지원하는 자체 UI 제작 툴입니다.<br>
UI를 비주얼화 시켜서 다양한 이벤트,함수, 동작 처리들을 진행할 수 있습니다.<br>
이에 관한 다양한 활용 방법은 공식 문서 검색을 통해 다양하게 찾아볼 수 있었습니다.

[UMG 공식 사이트 검색](https://www.unrealengine.com/ko/bing-search?x=0&y=0&filter=%EB%AC%B8%EC%84%9C&keyword=UMG)

이제 실습을 진행해보겠습니다.

## .Build.cs 설정

UMG를 사용하기 위해서 프로젝트에 모듈을 설정해주어야합니다. 이를 위해 프로젝트에 .Build.cs 파일을 설정해주어야합니다.<br>
저는 이전부터 진행하던 Prac 프로젝트에 계속 진행하므로 Prac.Build.cs 파일을 수정하게 됩니다.

### Prac.Build.cs

주석을 통해 어떠한 요소를 추가했고 어떤 줄의 코멘트를 해제했는지 표시해놓았습니다.

```C++
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;

public class Prac : ModuleRules
{
	public Prac(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

		//UMG 사용을 위해 종속성에 추가시켜 줍니다.
		PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "UMG" });

		PrivateDependencyModuleNames.AddRange(new string[] {  });

		// Uncomment if you are using Slate UI

		//코멘트 해제하여 프라이빗 모듈 목록에 "Slate"와 "SlateCore"를 추가해줍니다.
		PrivateDependencyModuleNames.AddRange(new string[] { "Slate", "SlateCore" });

		// Uncomment if you are using online features
		// PrivateDependencyModuleNames.Add("OnlineSubsystem");

		// To include OnlineSubsystemSteam, add it to the plugins section in your uproject file with the Enabled attribute set to true
	}
}
```

## PracGameModeBase 설정

메뉴 위젯을 교체하는 기능을 구현하기 위해서 PracGameModeBase에 ChangeMenuWidget이라는 함수를 구현해보겠습니다.

### PracGameModeBase.h

유저 위젯을 설정해주고 이 위젯을 교체하는 함수를 선언합니다.

```C++
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameModeBase.h"
#include "Blueprint/UserWidget.h"
#include "PracGameModeBase.generated.h"

/**
 *
 */
UCLASS()
class PRAC_API APracGameModeBase : public AGameModeBase
{
	GENERATED_BODY()

public:
	/** 현재 메뉴 위젯을 제거하고 지정된 클래스에서 새 메뉴 위젯을 만듭니다(제공된 경우). */
	// 유저 위젯을 새로 만들어서 기존 위젯을 대체하도록 하는 함수입니다.
	UFUNCTION(BlueprintCallable, Category = "UMG Game")
		void ChangeMenuWidget(TSubclassOf<UUserWidget> NewWidgetClass);

protected:
	/** Called when the game starts. */
	virtual void BeginPlay() override;


	/** 게임이 시작되면 메뉴로 사용할 위젯 클래스입니다. */
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "UMG Game")
		TSubclassOf<UUserWidget> StartingWidgetClass;

	/** 메뉴로 사용하고 있는 위젯 인스턴스입니다 */
	UPROPERTY()
		UUserWidget* CurrentWidget;
};

```

### PracGameModeBase.cpp

```C++
// Copyright Epic Games, Inc. All Rights Reserved.


#include "PracGameModeBase.h"

void APracGameModeBase::BeginPlay()
{
    Super::BeginPlay();
    //기존 시작 후 처리에 ChangeMenuWidget을 실행하여 메뉴 위젯을 변경하게 됩니다.
    ChangeMenuWidget(StartingWidgetClass);
}

void APracGameModeBase::ChangeMenuWidget(TSubclassOf<UUserWidget> NewWidgetClass)
{
    //CurrentWidget이 존재한다면 제거
    if (CurrentWidget != nullptr)
    {
        CurrentWidget->RemoveFromViewport();
        CurrentWidget = nullptr;
    }

    //NewWidgetClass가 존재한다면 이를 CurrentWidget에 할당
    if (NewWidgetClass != nullptr)
    {
        //CurrentWidget에 NewWidgetClass를 기반으로 한 새 <UUserWidget> 생성하여 할당
        CurrentWidget = CreateWidget<UUserWidget>(GetWorld(), NewWidgetClass);
        if (CurrentWidget != nullptr)
        {
            //CurrentWidget을 뷰포트에 등록
            CurrentWidget->AddToViewport();
        }
    }
}
```

## HowTo_UMGPlayerController

UI에 입력하기 위해 PlayerController를 설정해주어야합니다. 그렇기에 이를 상속받은
HowTo_UMGPlayerController 클래스를 만들어줍니다.

밑에 코드를 보면

```C++
SetInputMode(FInputModeGameAndUI());
```

문구를 통해서 UI에 입력할 수 있게 설정하는 클래스를 만들게 됩니다.

### HowTo_UMGPlayerController.h

```C++
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerController.h"
#include "HowTo_UMGPlayerController.generated.h"

/**
 *
 */
UCLASS()
class PRAC_API AHowTo_UMGPlayerController : public APlayerController
{
	GENERATED_BODY()

//BeginPlay() 함수 추가
public:
	virtual void BeginPlay() override;
};

```

### HowTo_UMGPlayerController.cpp

```C++
// Fill out your copyright notice in the Description page of Project Settings.


#include "HowTo_UMGPlayerController.h"

void AHowTo_UMGPlayerController::BeginPlay()
{
    Super::BeginPlay();
    SetInputMode(FInputModeGameAndUI());
}
```

## MainMenu(Widget Blueprint)

컨텐츠 브라우저에서 추가 버튼을 통해 Widget Blueprint를 생성해준 후 이름을 MainMenu라고 정해줍니다.<br>
이때 같은 방식으로 NewGameMenu도 하나 미리 만들어줍니다.

그 뒤 더블클릭해서 Blueprint를 열어줍니다. 그런 다음 버튼을 찾아서 추가해주고 버튼에 텍스트를 설정해줍니다.

![3](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/3.png)

다음 버튼에 OnClicked를 입력해주어야하는데 주의할 점은 해당 버튼을 변수 처리를 해주어야 Events가 활성화됩니다. 아래는 해당 설정입니다. 변수여부에 체크가 된 것을 확인해주셔야 합니다.

![4](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/4.png)

다음은 OnClicked(클릭 시) 이벤트를 추가 버튼을 누르고 해당 이벤트 처리를 블루프린트에서 세팅해줍니다.<br> 아래 사진과 같이 이벤트 세팅을 진행해줍니다.

마찬가지로 QuitButton을 생성해주고 이 버튼도 이벤트를 등록해줍니다.<br>
Quit Game은 자체적으로 지원해주는 함수이니 따로 구현할 필요는 없습니다.

![5](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/5.png)

## MenuPlayerController

Blueprint Class를 생성해주는데 부모 클래스로 일반 탭에 'PlayerController' 지정해주고 'MenuPlayerController' 를 생성해줍니다.

![6](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/6.png)

해당 MenuPlayerController를 연 뒤 아래 사진처럼 '마우스 커서 표시' 옵션을 켜줍니다.

![7](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/7.png)

## MenuGameMode

Blueprint Class를 생성해주는데 부모 클래스로 우리가 작성한 'PracGameModeBase'를 지정해주고 'MenuGameMode' 를 생성해줍니다.

![6](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/6.png)

다음 MenuGameMode를 설정해줍니다.

UMG Game 탭을 설정해주고 클래스 탭에서 '디폴트 폰 클래스' 와 '플레이어 컨트롤러 클래스'를 아래 사진처럼 세팅해줍니다.

![9](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/9.png)

다음은 프로젝트 세팅을 가서 기본 모드를 설정해줍니다.

![10](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/10.png)

## NewGameMenu

이제 기존 MenuGameMode에서 클릭하면 새롭게 바뀌게 될 NewGameMenu를 완성해보겠습니다.

아래 사진과 같이 TextBox 한 개와 Button 두 개를 배치합니다.

![17](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/17.png)

TextBox는 튜토리얼을 따라가던지 원하시는 배치대로 진행하셔도 무방합니다. 단, 변수 여부는 활성화해주셔야 합니다.

다음은 PlayGameButton을 설정해주어야 하는데 TextBox에 텍스트를 무언가 입력해야지만 활성화되는 기능을 넣으려 합니다.<br>
이를 위해 버튼에서 Behavior 탭에 활성화 여부에 바인드를 클릭해 추가해줍니다.

![11](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/11.png)

클릭 후 이벤트를 아래와 같이 설정합니다.

![12](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/12.png)

다음은 두 버튼의 클릭 이벤트를 생성해주고 사진처럼 세팅합니다.

![13](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/13.png)

## 실행결과 확인

이제 제대로 동작하는지 확인해보겠습니다. 사진으로 확인은 불가하지만 QuitButton 등도 다 동작하니 동작을 확인해줍니다.

### 시작화면

![14](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/14.png)

### NewGame 클릭 시

![14](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/14.png)

### TextBox 안에 텍스트 들어가면 버튼 활성화

![14](/Assets/Images/Unreal/실습/UMG와%20유저%20인터페이스/14.png)

## 마무리

이번에 공부했던 UMG와 인터페이스는 보통은 전문 디자이너가 담당하게 되는 영역이지만 이러한 영역도 동작 방식을 배워야 한다고 생각했기에 진행해보았습니다.

확실히 공부해보니 언리얼에서 UI는 어떻게 동작하는지 동작 방식을 이해할 수 있었고 또 프로젝트 세팅과 블루프린트를 통해 UI 자체적인 디자인을 쉽게 세팅할 수 있다는 점을 배웠습니다.

앞으로는 이제 개인적인 게임 프로젝트를 작업하게 될텐데 간단한 UI는 스스로 제작하고 진행시킬 수 있을 정도로 좀 더 숙련시키는 과정이 필요할 것 같습니다.
