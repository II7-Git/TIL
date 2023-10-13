# ToonTanks Project

툰탱크 프로젝트를 진행하면서 배우게 된 지식들을 간단히 기록해보겠다.

## 전방 선언

해당 내용에 대해서는 아래 링크에 적어놓았다.

[전방 선언 정리](/Unreal%20Engine/이론%20및%20정리/언리얼에서%20전방%20선언과%20사용%20이유.md)

## UPROPERTY 옵션

### 카테고리 지정

옵션에 Category="원하는 이름" 을 넣게 된다면
원하는 이름 카테고리 밑으로 변수가 들어가서 표시되게 된다. 이는 카테고리를 통한 변수 찾기 및 정리에 용이하기 때문에 유용하다

### 변수 노출

지금껏 습관처럼 사용하던 UPROPERTY에서 옵션인 EditAnywhere 같은 것에 실질적인 적용 예와 이벤트 그래프에서 사용할 수 있는 옵션에 대해서 알아보았다.

아래 표에 따라서 실제 사용 목적에 따라서 권한을 조절하는 식으로 작동시키면 된다.
![2](/Assets/Images/Unreal/실습/ToonTanks/2.png)

```C++
// 블루프린트와 인스턴스 어디서든 사용가능하며, 이벤트 그래프에서도 get,set이 가능하다
UPROPERTY(EditAnywhere, BlueprintReadWrite)
	float Speed = 400.0f;
```

### 만약 private에 속해있는 변수를 노출시키고 싶다면?

그럴 경우에는 UPROPERTY()에 meta=(AllowPrivateAccess = "true") 옵션을 넣어주면 된다.

### 최종 정리

![3](/Assets/Images/Unreal/실습/ToonTanks/3.png)

실제 예

```C++
private:
//블루프린트 ,인스턴스 둘 다 위치에서 보이기만 하며 이벤트 그래프에서 get이 가능하고 카테고리는 "Components" 밑에 소속되어있으며 private에 있어도 표시가 가능하다
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components", meta = (AllowPrivateAccess = "true"))
	class UCapsuleComponent *CapsuleComp;
```

해당 기능을 사용하여 실제 블루프린트로 탱크와 터렛을 만들어 배치시킨 모습
![3](/Assets/Images/Unreal/실습/ToonTanks/4.png)

## 플레이 할 Tank 만들기

### 스프링 암과 카메라 달아주기

![5](/Assets/Images/Unreal/실습/ToonTanks/5.png)

이번엔 위에 사진과 같이 플레이 할 Tank 액터에 스프링 암과 카메라를 달아준다.

```C++
ATank::ATank()
{
    SpringArm = CreateDefaultSubobject<USpringArmComponent>(TEXT("Spring Arm"));
    SpringArm->SetupAttachment(RootComponent);

    Camera = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
    Camera->SetupAttachment(SpringArm);
}
```

주의할 점으로는 SpringArm에 달린 카메라의 위치를 수정하는 일은 보통은 하지 않는다. <br>이유로는 SpringArm에 내장된 기능들로 벽을 만나면 카메라가 벽을 넘지 않게 거리를 수정해주거나 각도를 트는 등에 기능들이 구현되어있는데 그 끝에 달린 카메라를 임의로 수정하면 SpringArm을 사용함으로써 노리는 기능들이 정상 동작하지 않기 때문이다.

그렇기에 SpringArm에 위치만 수정하여 원하는 시점을 연출하자.

### WASD 이동 처리

이동 처리는 프로젝트 세팅에서 해놓은 BindAxis를 각각 Move,Turn 함수와 매핑시켜서 동작시키는데 그 과정에서는 이전에도 써본 <br>
SetupPlayerInputComponent(UInputComponent \*PlayerInputComponent)<br>
함수를 사용한다.

실제 코드를 보면

```C++
void ATank::SetupPlayerInputComponent(UInputComponent *PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    PlayerInputComponent->BindAxis(TEXT("MoveForward"), this, &ATank::Move);
    PlayerInputComponent->BindAxis(TEXT("Turn"), this, &ATank::Turn);
}
void ATank::Move(float Value)
{
    FVector DeltaLocation = FVector::ZeroVector;
    // X = Value* DeltaTime*Speed
    DeltaLocation.X = Value * Speed * UGameplayStatics::GetWorldDeltaSeconds(this);
    AddActorLocalOffset(DeltaLocation, true);
}

void ATank::Turn(float Value)
{
    FRotator DeltaRotation = FRotator::ZeroRotator;
    // Yaw = Value * DeltaTime * TurnRate
    DeltaRotation.Yaw = Value * TurnRate * UGameplayStatics::GetWorldDeltaSeconds(this);
    AddActorLocalRotation(DeltaRotation, true);
}
```

위와 같은데 먼저 SetupPlayerInputComponent() 함수에서 BindAxis함수로 축매핑에서 각각의 이름을 가진 요소의 값을 가져와서 이 클래스에 있는 함수 Move,Turn에 각각 값을 전달하여 실행한다.

그러면 각 Move, Turn 에서는 Value 인자로 전달받은 값을 가지고 이동을 처리하게 되는데 이동을 처리할 때 Actor각각의 지역 좌표계로 이동을 처리하기 위해 AddActorLocalOffset(),AddActorLocalRotation()를 사용한다.

또 일관된 속도를 사용하기 위해서 DeltaSecond가 필요한데 그 값은 게임에서 사용되는 여러 통계를 모아놓은 UGameplayStatics을 이용한다.

[UGameplayStatics::GetWorldDeltaSeconds()](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Kismet/UGameplayStatics/GetTimeSeconds/)

또 AddActorLocalOffset(),AddActorLocalRotation()의 함수를 보면 bool값이 있는 것을 확인할 수 있는데 bSweep값을 의미한다. 기본은 False이고 이 값은 해당 Actor가 다른 콜리전과 충돌할 때 겹치는 것을 허용할 것인지 아니면 그 전에 멈추게 할 것인지를 정하는 인자값이다.<br>
그래서 true를 하게 되면 콜리전 프리셋에 설정한대로 콜리전을 구분해서 충돌하면 그 전 위치에서 멈추게 구현이 된다.

이러한 기능들을 사용해서 이동을 구현했다.

### Turret 회전 처리

플레이 할 탱크와 상대가 될 타워에는 각각 미사일을 쏠 터렛이 존재하는데 플레이어는 이를 마우스 위치에 맞게 타워는 플레이어의 탱크에 위치에 맞게 회전해야할 필요가 있다.<br>
그래서 회전 자체는 공통된 기능이고 TurretMesh를 사용하기에 공통된 함수로 BasePawn에서 회전을 구현한다.

회전 함수

```C++
void ABasePawn::RotateTurret(FVector LookAtTarget)
{
	FVector ToTarget = LookAtTarget - TurretMesh->GetComponentLocation();
	FRotator LookAtRotation = FRotator(0.f, ToTarget.Rotation().Yaw, 0.f);
	TurretMesh->SetWorldRotation(FMath::RInterpTo(TurretMesh->GetComponentRotation(), LookAtRotation, UGameplayStatics::GetWorldDeltaSeconds(this), 25.f));
}
```

보면 급격한 회전을 방지하기 위해 FMath::RinterpTo() 함수로 보간 처리를 해서 부드럽게 회전하게 의도했고 회전하는 LookAtRotation을 보면 Yaw값만 받아서 위아래로 회전을 막는다.

이를 바탕으로 Tank와 Tower에서 Tick() 안에서 각각 사용하여 원하는 기능을 구현한다

Tank.cpp

```C++
if (PlayerControllerRef)
    {
        FHitResult HitResult;

        PlayerControllerRef->GetHitResultUnderCursor(
            ECollisionChannel::ECC_Visibility,
            false,
            HitResult);

        RotateTurret(HitResult.ImpactPoint);
    }
```

마우스 위치에 값을 HitResult에 넣고 그 값을 바탕으로 회전

Tower.cpp

```C++

if (Tank)
    {
        float Distance = FVector::Dist(GetActorLocation(), Tank->GetActorLocation());

        if (Distance <= FireRange)
        {
            RotateTurret(Tank->GetActorLocation());
        }
    }
```

타워의 사거리 안에 있다면 탱크에 위치로 회전

---

이를 바탕으로 실행해보면 원하는 대로 구현된 것을 볼 수 있다.

탱크의 터렛 회전
![6](/Assets/Images/Unreal/실습/ToonTanks/6.png)

타워의 터렛 회전
![7](/Assets/Images/Unreal/실습/ToonTanks/7.png)
