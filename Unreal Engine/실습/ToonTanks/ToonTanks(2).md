# 플레이 할 Tank와 Tower 만들기

## 스프링 암과 카메라 달아주기

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

## WASD 이동 처리

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

## Turret 회전 처리

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

## Tower의 발사 타이머 설정

타워는 일정 시간의 쿨타임을 가지고 사거리 안에 Tank가 있다면 미사일을 발사한다.

이때 사용하는 것이 FTimerManager이다.

### [FTimerManager DOCS](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/FTimerManager/)

FTimerHandle 객체를 사용해서 컨트롤을 하며 FTimerManager.SetTimer()를 이용해서 지정 시간마다 콜 백 함수를 발동시키는 기능을 구현한다.

아래는 Tower에서 구현한 SetTimer()기능이다. 위에 주석으로 설명을 달아놓았다.

```C++
void ATower::BeginPlay()
{
    Super::BeginPlay();

    Tank = Cast<ATank>(UGameplayStatics::GetPlayerPawn(this, 0));

    // GetWorldTimerManager().SetTimer(FTimerHandle & InOutHandle,UserClass * InObj, InTimerMethod, float InRate, bool InbLoop)
    // GetWorldTimerManager().SetTimer(FTimerHandle,실행할 객체(this),실행할 함수의 주소,이 시간이 지나면 콜백 함수 호출,함수를 반복 실행할지 여부)
    GetWorldTimerManager().SetTimer(FireRateTimerHandle, this, &ATower::CheckFireCondition, FireRate, true);
}
```

## Projectile (발사체) 설정

탱크와 타워는 동일한 미사일을 발사하기에 BasePawn에서 Fire()에서 발사될 발사체를 만들어야한다. 이를 위해 Projectile 클래스 만들고 이를 파생시킨 블루프린트 클래스를 만들어서 StaticMesh에 총알을 등록해준다.
![11](/Assets/Images/Unreal/실습/ToonTanks/11.png)

## SpawnActor

발사체를 블루프린트로 만들었으니 이를 실제 액터로 소환시켜야한다.

이때 Projectile을 TSubClassOf<> 로 등록해준다.TSubClassOf<>는 UClass를 지정해주는 것이 가능하다.

```C++
UPROPERTY(EditDefaultsOnly, Category = "Combat")
	TSubclassOf<class AProjectile> ProjectileClass;
```

### 왜 UClass?

[TSubClassOf Docs](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/ProgrammingWithCPP/UnrealArchitecture/TSubclassOf/)

UClass는 리플렉션 기능이 내장되어 있어서 C++와 블루프린트 사이에 정보가 교환이 가능하기에 UClass를 사용한다.

그래서 왜 사용하나면 SpawnActor<C++ Class>(UClass,FVector,FRotator)가 UWorld에서 구현되어 있고 우리는 우리가 원하는 설정이 완료된 (여기서는 스태틱 메시에 총알을 적용시킨) 블루프린트 클래스를 소환하고 싶은데 이는 C++클래스 자체에선 블루프린트 클래스를 지정할 방법이 없다. 그래서 블루프린트 클래스를 부르기 위해서 해당 정보를 교환하는 UClass를 사용한다. 그렇기에 TSubClassOf<>()를 사용한다.

### SpawnActor 적용 모습

![8](/Assets/Images/Unreal/실습/ToonTanks/8.png)

## ProjectileMovementComponent를 통해 발사체 이동

Projectile(발사체)를 이동시킬 방법이 여러개 있지만 이번에는 ProjectileMovementComponent를 이용한다.

### [ProjectileMovementComponent](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/GameFramework/UProjectileMovementComponent/)

```C++
ProjectileMovementComponent = CreateDefaultSubobject<UProjectileMovementComponent>(TEXT("Projectile Movement Component"));
	ProjectileMovementComponent->MaxSpeed = 1300.f;
	ProjectileMovementComponent->InitialSpeed = 1300.f;
```

MaxSpeed랑 InitialSpeed만 지정해주어서 간단히 구현

## 발사체 타격(Hit) 이벤트

### [OnComponentHit](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Components/UPrimitiveComponent/OnComponentHit/)

### [FComponentHitSignature](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Components/FComponentHitSignature/)

위 요소들을 사용한다.

### Multicast Delegate

OnComponentHit은 Multicast Delegate라고 하는데 이는 여러 개의 함수를 바인딩할 수 있다는 것을 뜻한다. 그래서 여러 이벤트를 리스트 형태로 등록해서 전부 실행시킬 수 있다.

대략 순서는 아래와 같다

> - Invocation List에 함수들 등록
> - 발사체가 무언가를 쳐서 Hit이벤트가 발생하면 델리케이트는 바인딩 된 함수를 가진 모든 클래스에 Broadcast를 보냄
> - Broadcast를 받은 클래스들은 전부 해당 함수를 실행시킨다.

### 실제 바인딩하기

OnComponentHit.AddDynamic()을 사용하여 함수를 델리케이드에 바인딩한다.

```C++
void AProjectile::BeginPlay()
{
	Super::BeginPlay();

	ProjectileMesh->OnComponentHit.AddDynamic(this, &AProjectile::OnHit);
}
```

이때 OnHit의 인자들이 정확한 FComponentHitSignature의 요소여야한다.<br>
아래는 FComponentHitSignature의 정의 중 일부분을 가져온 것인데 이것과 같은 순서로 인자의 타입이 배치되어있어야한다

```C++
/**
 * Delegate for notification of blocking collision against a specific component.
 * NormalImpulse will be filled in for physics-simulating bodies, but will be zero for swept-component blocking collisions.
 */
DECLARE_DYNAMIC_MULTICAST_SPARSE_DELEGATE_FiveParams( FComponentHitSignature, UPrimitiveComponent, OnComponentHit, UPrimitiveComponent*, HitComponent, AActor*, OtherActor, UPrimitiveComponent*, OtherComp, FVector, NormalImpulse, const FHitResult&, Hit );
/** Delegate for notification of start of overlap with a specific component */
DECLARE_DYNAMIC_MULTICAST_SPARSE_DELEGATE_SixParams( FComponentBeginOverlapSignature, UPrimitiveComponent, OnComponentBeginOverlap, UPrimitiveComponent*, OverlappedComponent, AActor*, OtherActor, UPrimitiveComponent*, OtherComp, int32, OtherBodyIndex, bool, bFromSweep, const FHitResult &, SweepResult);
/** Delegate for notification of end of overlap with a specific component */
DECLARE_DYNAMIC_MULTICAST_SPARSE_DELEGATE_FourParams( FComponentEndOverlapSignature, UPrimitiveComponent, OnComponentEndOverlap, UPrimitiveComponent*, OverlappedComponent, AActor*, OtherActor, UPrimitiveComponent*, OtherComp, int32, OtherBodyIndex);

```

아래는 OnHit의 인자에 FComponentHitSignature구조에 맞춰 매개변수를 넣고 이름은 원하는대로 지은 것이다. 즉, 매개변수의 구조만 정확히 맞추면 된다.

```C++
UFUNCTION()
	void OnHit(UPrimitiveComponent *HitComp, AActor *OtherActor, UPrimitiveComponent *OtherComp, FVector NormalImpulse, const FHitResult &Hit);
```
