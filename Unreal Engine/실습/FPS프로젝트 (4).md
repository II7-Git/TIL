# FPS 프로젝트 정리

FPS프로젝트를 진행하면서 새롭게 배웠던 내용들을 쭉 정리해보았습니다.

[FPS프로젝트 (1)](<./FPS프로젝트%20(1).md>)<br>
[FPS프로젝트 (2)](<./FPS프로젝트%20(2).md>)<br>
[FPS프로젝트 (3)](<./FPS프로젝트%20(3).md>)<br>

이번엔 위에 프로젝트를 진행하면서 새롭게 접했던 함수나 기능들을 한번 쭉 정리해보려고 합니다.

## 클래스 및 함수 정리

프로젝트를 진행하면서 알게된 클래스와 함수 등을 정리해보겠습니다.<br>
클래스 별로 묶어서 기록하겠습니다.<br>
함수는 원형을 적고 구현부 코드를 적어서 실제 사용에 용이하게 적어보겠습니다.

---

### AGameModeBase

이 클래스를 상속 받아서 게임 모드에 원하는 추가 내용을 구현하여 프로젝트 세팅에서 게임 모드를 대신합니다.

#### - virtual void StartPlay() override

게임 모드에서 처음 시작할 때 필요한 기능을 구현하여 오버라이드할 때 사용합니다.

```c++
void AFPSProjectGameModeBase::StartPlay() {
	Super::StartPlay();

}
```

### GEngine

프로젝트 전역에 선언되어 있는 엔진 포인터입니다. 엔진이 동작하는지 여부를 확인하고 엔진 자체에서 제공되는 기능들을 사용할 수 있지만 엔진이 동작하는지 꼭 확인하고 기능을 써야합니다.

#### - AddOnScreenDebugMessage()

화면에 특정 위치에 디버그 메시지를 표시하게 됩니다.<br>
다양한 인수를 바탕으로 여러 옵션을 제공합니다.

```c++
if (GEngine) {
		// 디버그 메시지를 5 초간 표시합니다.
		// "키" (첫 번째 인수) 값을 -1 로 하면 이 메시지를 절대 업데이트하거나 새로고칠 필요가 없음을 나타냅니다.
		GEngine->AddOnScreenDebugMessage(-1, 5.0f, FColor::Yellow, TEXT("Hello World, this is FPSGameMode!"));
	}
```

---

### ACharacter

플레이어가 조종할 캐릭터가 주로 상속받을 부모 클래스로 다양한 기본 제공 기능들이 있습니다.

#### -BeginPlay()

게임이 시작되고 난 뒤 한번만 동작할 내용들을 작성합니다.

```c++
void AFPSCharacter::BeginPlay()
{
	Super::BeginPlay();

}

```

#### -Tick(float DeltaTime)

매 프레임마다 호출되는 함수입니다.

```c++
// Called every frame
void AFPSCharacter::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

}
```

#### -SetupPlayerInputComponent(UInputComponent\* PlayerInputComponent)

클라이언트에 프로젝트 세팅을 매핑해놓은 축,액션 매핑 등의 동작이 입력되면 호출되는 함수로 UInputComponent를 통해서 어떠한 매핑이 동작했는지 정보가 들어오고 이를 현재 클래스의 구현해놓은 함수와 바인딩하여 해당 함수가 동작할 수 있게 해줍니다.

```C++
// Called to bind functionality to input
void AFPSCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

PlayerInputComponent->BindAxis("MoveForward", this, &AFPSCharacter::MoveForward);
PlayerInputComponent->BindAction("Fire", IE_Pressed, this, &AFPSCharacter::Fire);
}
```

### UInputComponent

위에서 설명됐듯 프로젝트에서 설정한 축, 액션 매핑 등의 정보를 받아서 처리하게 됩니다.

#### -BindAxis<AFPSCharacter>(FName AxisName, AFPSCharacter *Object, void (    AFPSCharacter::*Func)(float))

#### -BindAction<AFPSCharacter>(FName ActionName, EInputEvent KeyEvent,     AFPSCharacter *Object, void (AFPSCharacter::*Func)())

```C++
//MoveForward라고 명명된 축 매핑이 입력되면 MoveForward 함수에 값을 넘기면서 동작시켜라
PlayerInputComponent->BindAxis("MoveForward", this, &AFPSCharacter::MoveForward);
//Fire라고 액션 매핑된 버튼이 눌리게 된다면(IE_Pressed) Fire() 함수를 동작시켜라
PlayerInputComponent->BindAction("Fire", IE_Pressed, this, &AFPSCharacter::Fire);
```

### UCameraComponent

카메라 관련된 기능들을 지원하며 해당 컴포넌트를 사용하여 카메라를 특정 컴포넌트에 붙이거나 위치를 수정하는 등의 기능들을 지원합니다.

#### - SetupAttachment(USceneComponent \*InParent, FName InSocketName = NAME_None);

특정 컴포넌트에 카메라를 붙인다. 해당 컴포넌트와 같이 이동하게 된다.

```C++
FPSCameraComponent->SetupAttachment(GetCapsuleComponent());
```

#### - SetRelativeLocation(FVector NewLocation, bool bSweep = false, FHitResult _OutSweepHitResult    = (FHitResult _)nullptr, ETeleportType Teleport = ETeleportType::None)

카메라의 상대적인 위치를 조정하게됩니다.

```C++
FPSCameraComponent->SetRelativeLocation(FVector(0.0f, 0.0f, 50.0f + BaseEyeHeight));
```

#### - bUsePawnControlRotation

변수로써 폰이 로테이션을 제어하는 것을 허락합니다.

```C++
FPSCameraComponent->bUsePawnControlRotation = true;
```

### USkeletalMeshComponent

메시 컴포넌트를 다룰 때 사용하게 됩니다

#### - SetOnlyOwnerSee(bool bNewOnlyOwnerSee)

해당 메시 컴포넌트를 소유하고 있는 오브젝트만 이 메시 컴포넌트를 보게 됩니다

```C++
FPSMesh->SetOnlyOwnerSee(true);
```

#### - SetupAttachment(USceneComponent \*InParent, FName InSocketName = NAME_None)

메시 컴포넌트를 특정 컴포넌트에 붙입니다.

```C++
FPSMesh->SetupAttachment(FPSCameraComponent);
```

#### - bCastDynamicShadow

#### - CastShadow

해당 메시의 그림자 캐스팅 옵션들입니다. 필요에 따라 조절합니다.

```C++
// 일부 환경 섀도잉을 꺼서 메시가 하나처럼 보이는 느낌을 보존합니다.
	FPSMesh->bCastDynamicShadow = false;
	FPSMesh->CastShadow = false;
```

### UPrimitiveComponent

#### - SetOwnerNoSee(bool bNewOwnerNoSee)

```C++
// 소유 플레이어는 일반 (삼인칭) 바디 메시를 보지 못합니다.
	GetMesh()->SetOwnerNoSee(true);
```

### ProjectileMovementComponent

발사체에 주로 사용되는 클래스입니다.<br>
발사체에 관련된 다양한 옵션들을 지원합니다.

#### - SetUpdatedComponent(USceneComponent \*NewUpdatedComponent)

인자 컴포넌트를 발사체로 등록하게 됩니다.

```C++
ProjectileMovementComponent->SetUpdatedComponent(CollisionComponent);
```

#### - InitialSpeed : 발사체의 속도(float)

#### - MaxSpeed : 발사체의 최대 속도 제한(float)

#### - bRotationFollowsVelocity : true일시 발사체가 이동 방향에 맞춰 매 프레임 회전

#### - bShouldBounce : true일 시 바운스가 동작해서 발사체가 튕기게 된다.

여러가지 발사체 관련 변수 옵션들입니다.

```C++
	ProjectileMovementComponent->InitialSpeed = 3000.0f;
	ProjectileMovementComponent->MaxSpeed = 3000.0f;
	ProjectileMovementComponent->bRotationFollowsVelocity = true;
	ProjectileMovementComponent->bShouldBounce = true;
	ProjectileMovementComponent->Bounciness = 0.3f;
```

### AActor

#### - InitialLifeSpan

해당 액터가 이 변수 시간만큼 유지되고 죽게됩니다.
0이면 영구히 지속됩니다.

```C++
InitialLifeSpan = 3.0f;
```

## 끝마치며

이번 프로젝트에서도 여러 기능들을 보았지만 언리얼이 가지는 가장 큰 힘은 이러한 게임에서 주로 쓰이는 기능들을 기본적인 바탕 클래스로 제공하고 이를 상속받아서 필요한 기능들을 오버라이드해서 쓸 수 있는 점이라는 생각이 들었습니다. 다른 다양한 프로젝트를 통해서 보다 많은 기능들을 더 다뤄보도록 하겠습니다.
