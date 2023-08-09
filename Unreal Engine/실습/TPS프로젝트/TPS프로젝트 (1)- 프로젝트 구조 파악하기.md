# TPS프로젝트 (1)- 프로젝트 구조 파악하기

게임을 개발하기 위해 TPS 프로젝트의 기본적인 구조를 파악해보고 이를 토대로 좀 더 해당 프로젝트를 발전 시켜보려하고 이 과정을 기록하려 합니다.

## TPS 프로젝트 생성

버전은 Unreal Engine 5.2.0 에서 진행되었고 해당 버전에서 프로젝트 생성-> 게임 -> 삼인칭 프로젝트 순으로 프로젝트를 생성해줍니다.

그러고 실행을 해보면 아래와 같은 프로젝트 동작을 확인할 수 있습니다.

![1](/Assets/Images/Unreal/실습/TPSProject/1.기존프로젝트분석/1.png)
(자유 시점 관찰)

![1](/Assets/Images/Unreal/실습/TPSProject/1.기존프로젝트분석/2.png)
(플레이어 카메라 시점)

오늘부터는 이 프로젝트를 토대로 코드와 동작을 분석하여 해당 프로젝트의 캐릭터 동작을 추가해보는 식으로 프로젝트를 학습하고 발전시켜 나가보려 합니다.

일단 생각하고 있는 추가 기능은 '뛰기, 걷기 모션 구현' ,'파쿠르 기능 구현(애니메이션 X)' 정도 등으로 우선하여 구현하려 합니다.

## 기본 프로젝트 분석

먼저 해당 코드들을 봐보니 'TPSProjectCharacter' 라는 C++ 클래스를 바탕으로 'BP_ThirdPersonCharacter'라는 블루프린트를 만들어 이를 플레이시 생성하여 동작시키는 구조를 확인할 수 있었습니다.

그럼 이제 'TPSProjectCharacter' 부터 하나씩 학습해보겠습니다.

먼저 코드에 대해 학습한 내용을 주석을 통해 기록해보았습니다.

### TPSProjectCharacter.h

```C++
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "InputActionValue.h"
#include "TPSProjectCharacter.generated.h"


UCLASS(config=Game)
class ATPSProjectCharacter : public ACharacter
{
	GENERATED_BODY()

	// 캐릭터의 등록할 카메라 요소들과 액션 요소들이 따로 존재하기에 이를 변수 선언하고 Blueprint에서 매핑하기에 BluePrintOnly를 사용하여 UPROPERTY로 등록
	/** Camera boom positioning the camera behind the character */
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = Camera, meta = (AllowPrivateAccess = "true"))
	class USpringArmComponent* CameraBoom;

	/** Follow camera */
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = Camera, meta = (AllowPrivateAccess = "true"))
	class UCameraComponent* FollowCamera;

	// 이 DefaultMappingContext에 매핑된 요소를 확인하면 그 쪽에서 키 매핑이 전부 되어있는 것을 확인할 수 있다.
	/** MappingContext */
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Input, meta = (AllowPrivateAccess = "true"))
	class UInputMappingContext* DefaultMappingContext;

	/** Jump Input Action */
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Input, meta = (AllowPrivateAccess = "true"))
	class UInputAction* JumpAction;

	/** Move Input Action */
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Input, meta = (AllowPrivateAccess = "true"))
	class UInputAction* MoveAction;

	/** Look Input Action */
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Input, meta = (AllowPrivateAccess = "true"))
	class UInputAction* LookAction;

public:
	ATPSProjectCharacter();


// 플레이어가 사용할 함수는 Input값등을 외부에서 받아서 내부로직으로만 처리하기에 protected로 선언한 것을 확인
protected:

	/** Called for movement input */
	void Move(const FInputActionValue& Value);

	/** Called for looking input */
	void Look(const FInputActionValue& Value);


protected:
	// APawn interface
	virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;

	// To add mapping context
	virtual void BeginPlay();

public:
	//Force Inline은 말에서 볼 수 있듯 강제로 함수를 Inline화 시키는 것이다. //이렇게 되면 함수 바디를 컴파일 타임에 진행하여 오버헤드가 줄어들게 된다.
	/** Returns CameraBoom subobject **/
	FORCEINLINE class USpringArmComponent* GetCameraBoom() const { return CameraBoom; }
	/** Returns FollowCamera subobject **/
	FORCEINLINE class UCameraComponent* GetFollowCamera() const { return FollowCamera; }
};
```

### TPSProjectCharacter.cpp

```C++
// Copyright Epic Games, Inc. All Rights Reserved.

#include "TPSProjectCharacter.h"
#include "Camera/CameraComponent.h"
#include "Components/CapsuleComponent.h"
#include "Components/InputComponent.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "GameFramework/Controller.h"
#include "GameFramework/SpringArmComponent.h"
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"


//////////////////////////////////////////////////////////////////////////
// ATPSProjectCharacter

ATPSProjectCharacter::ATPSProjectCharacter()
{
	// 충돌을 감지할 캡슐 콜리전의 크기를 조절
	// Set size for collision capsule
	GetCapsuleComponent()->InitCapsuleSize(42.f, 96.0f);

	// 컨트롤러가 회전한다고 해서 진짜 회전하지 못하게하기 , 이를 통해 카메라에 영향을 막음
	// 실제 true로 값을 고쳐 확인해보니 플레이어 캐릭터가 마우스 이동에 따라 플레이어 오브젝트가 같이 움직여서 캐릭터 이동에 적합하지 않은 것을 확인
	// Don't rotate when the controller rotates. Let that just affect the camera.
	bUseControllerRotationPitch = false;
	bUseControllerRotationYaw = false;
	bUseControllerRotationRoll = false;

	// Configure character movement
	// 캐릭터 무브먼트 가져와서 설정

	//자동으로 캐릭터 이동방향을 움직이는 방향에 맞춰준다.
	GetCharacterMovement()->bOrientRotationToMovement = true; // Character moves in the direction of input...
	//회전 속도로 Y값이 증가할수록 회전 방향으로 빠르게 움직인다.
	GetCharacterMovement()->RotationRate = FRotator(0.0f, 500.0f, 0.0f); // ...at this rotation rate

	// Note: For faster iteration times these variables, and many more, can be tweaked in the Character Blueprint
	// instead of recompiling to adjust them

	//캐릭터 점프 수치
	GetCharacterMovement()->JumpZVelocity = 700.f;
	//공중에 있을 때 캐릭터 이동가능한 수치
	GetCharacterMovement()->AirControl = 0.35f;
	//최대 걷는 속도
	GetCharacterMovement()->MaxWalkSpeed = 500.f;
	// 스틱 조종에서 최소한으로 걷는 속도
	GetCharacterMovement()->MinAnalogWalkSpeed = 20.f;
	// 걷는 것에 마찰력 높을수록 급정거하게 된다.
	GetCharacterMovement()->BrakingDecelerationWalking = 2000.f;

	// Create a camera boom (pulls in towards the player if there is a collision)
	// 카메라암 설정
	CameraBoom = CreateDefaultSubobject<USpringArmComponent>(TEXT("CameraBoom"));
	CameraBoom->SetupAttachment(RootComponent);
	CameraBoom->TargetArmLength = 400.0f; // The camera follows at this distance behind the character
	//캐릭터에 따라 회전
	CameraBoom->bUsePawnControlRotation = true; // Rotate the arm based on the controller

	// Create a follow camera
	//카메라 설정
	FollowCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("FollowCamera"));
	FollowCamera->SetupAttachment(CameraBoom, USpringArmComponent::SocketName); // Attach the camera to the end of the boom and let the boom adjust to match the controller orientation
	//카메라암에 붙어있기에 캐릭터 회전에 따라 이동하지 못하게 함
	FollowCamera->bUsePawnControlRotation = false; // Camera does not rotate relative to arm

	// Note: The skeletal mesh and anim blueprint references on the Mesh component (inherited from Character)
	// are set in the derived blueprint asset named ThirdPersonCharacter (to avoid direct content references in C++)
}

void ATPSProjectCharacter::BeginPlay()
{
	// Call the base class
	Super::BeginPlay();

	//Add Input Mapping Context
	//Input Context 등록
	if (APlayerController* PlayerController = Cast<APlayerController>(Controller))
	{
		if (UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PlayerController->GetLocalPlayer()))
		{
			Subsystem->AddMappingContext(DefaultMappingContext, 0);
		}
	}
}

//////////////////////////////////////////////////////////////////////////
// Input

void ATPSProjectCharacter::SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent)
{
	// 동작들 등록
	// Set up action bindings
	if (UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(PlayerInputComponent)) {

		//Jumping
		EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Triggered, this, &ACharacter::Jump);
		EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Completed, this, &ACharacter::StopJumping);

		//Moving
		EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &ATPSProjectCharacter::Move);

		//Looking
		EnhancedInputComponent->BindAction(LookAction, ETriggerEvent::Triggered, this, &ATPSProjectCharacter::Look);

	}

}

//이동 구현
void ATPSProjectCharacter::Move(const FInputActionValue& Value)
{
	// input is a Vector2D
	FVector2D MovementVector = Value.Get<FVector2D>();

	if (Controller != nullptr)
	{
		// find out which way is forward
		//카메라 시점으로 정면 구하기
		//이를 통해 이동 방향들 구해서 MovementInput값에 등록
		const FRotator Rotation = Controller->GetControlRotation();
		const FRotator YawRotation(0, Rotation.Yaw, 0);

		// get forward vector
		const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);

		// get right vector
		const FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);

		// add movement
		AddMovementInput(ForwardDirection, MovementVector.Y);
		AddMovementInput(RightDirection, MovementVector.X);
	}
}

//마우스 시점 구현
void ATPSProjectCharacter::Look(const FInputActionValue& Value)
{
	// input is a Vector2D
	FVector2D LookAxisVector = Value.Get<FVector2D>();

	if (Controller != nullptr)
	{
		// add yaw and pitch input to controller
		AddControllerYawInput(LookAxisVector.X);
		AddControllerPitchInput(LookAxisVector.Y);
	}
}
```

### 추가 분석

위 코드를 바탕으로 블루프린트 캐릭터를 만든 후 무브먼트 입력 등을 등록해 구현해놓은 모습을 공부할 수 있었습니다.

블루프린트 캐릭터에 모습- 입력, 캐릭터 메시 등의 에셋이 이곳에서 등록된 것을 확인할 수 있었습니다.
![3](/Assets/Images/Unreal/실습/TPSProject/1.기존프로젝트분석/3.png)

인풋 컨텍스트에 등록된 요소의 모습
![4](/Assets/Images/Unreal/실습/TPSProject/1.기존프로젝트분석/4.png)

위에 사진을 보면 WASD를 IE_Move 하나에 등록해놓은 모습을 볼 수 있었습니다. 그래서 WASD값을 어떻게 구분하는지 알아보았습니다.

그 값은 Modifiers를 통해서 구현해놓았었습니다.<br>
아래값을 보면 W,S는 '스위즐 입력 축 값'이라는 옵션이 있는데 이는 Y축 값으로 변경시켜주는 인덱스 값이었습니다.<br>
또 A,S 에 있는 'Negate' 옵션은 값을 -로 바꿔주는 옵션으로 이를 통해 반대 방향 값을 구현한 것이었습니다.
![5](/Assets/Images/Unreal/실습/TPSProject/1.기존프로젝트분석/5.png)

## 마무리

코드를 분석해보니 막상 어려운 기능들은 없지만 Unreal 상에서 기능들을 활용해 블루프린트 등 연결된 기능이 좀 어렵게 느껴졌습니다.

그래도 조금 더 공부하고 이 프로젝트를 발전시키면서 더 학습해보고 싶습니다.
