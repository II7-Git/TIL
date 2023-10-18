# ToonTanks(3)

## HealthComponent - 체력 기능 만들기

## 언리얼 데미지 시스템

### OnTakeAnyDamage

언리얼에 모든 액터는 데미지 이벤트를 상속 받는다.<br>
OnTakeAnyDamage() 라고 하는데 이전에 썼던 OnComponentHit()과 같이 멀티캐스트 델리게이트로서 발동하면 해당되는 객체들에게 DamageTaken() 콜백 함수를 발동시킨다. 이를 통해서 데미지 처리를 구현한다.<br>
이때 데미지의 적용 시키는 방법이 필요한데 [UGameplayStatics::ApplyDamage()](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Kismet/UGameplayStatics/ApplyDamage/)를 사용하면 된다. ApplyDamage()의 인풋 타입을 보면 DamageTaken()에서 얻는 변수들과 같은 것을 볼 수 있고 이를 이용하면 된다.

### 구현

#### Projectile에서 OnHit

OnHit에서는 받은 정보들을 가지고 ApplyDamage()에 넣을 변수들을 생성하고 ApplyDamage를 실행한다.

```C++
void AProjectile::OnHit(UPrimitiveComponent *HitComp, AActor *OtherActor, UPrimitiveComponent *OtherComp, FVector NormalImpulse, const FHitResult &Hit)
{
	auto MyOwner = GetOwner();
	if (MyOwner == nullptr)
		return;

	auto MyOwnerInstigator = MyOwner->GetInstigatorController();

	auto DamageTypeClass = UDamageType::StaticClass();

	// 충돌한 상대가 있고 그게 자신이 아니며, 자신의 소유자도 아닐 때 데미지 이벤트 발동
	if (OtherActor && OtherActor != this && OtherActor != MyOwner)
	{
		UGameplayStatics::ApplyDamage(OtherActor, Damage, MyOwnerInstigator, this, DamageTypeClass);
		Destroy();
	}
}

```

#### HealthComponent

체력을 관리하는 HealthComponent에서는 BeginPlay에서는 OnTakeAnyDamage에 데미지를 처리할 함수를 바인딩해준다

```C++
// Called when the game starts
void UHealthComponent::BeginPlay()
{
	Super::BeginPlay();
	Health = MaxHealth;

	// OnTakeAnyDamage()에 바인딩해서 델리케이트의 인보케이션 리스트에 DamageTaken함수 추가
	GetOwner()->OnTakeAnyDamage.AddDynamic(this, &UHealthComponent::DamageTaken);
}

void UHealthComponent::DamageTaken(AActor *DamagedActor, float Damage, const UDamageType *DamageType, AController *Instigator, AActor *DamageCauser)
{
	if (Damage <= 0.f)
		return;

	Health -= Damage;
	UE_LOG(LogTemp, Warning, TEXT("Health: %f"), Health);
    //여기서 데미지에 따른 죽음 처리할 것
}

```

#### 결과

OnHit()이 발생하면 필요한 처리 후 ApplyDamage()를 발생시키는데 이러면 OnTakeAnyDamage에 인보케이션 리스트에 바인딩 된 함수들이 실행된다. 이를 통해서 발사체와 충돌한 객체들의 데미지 처리 콜백 함수가 실행된다.

### 죽음 처리

#### Tank

플레이하는 탱크같은 경우는 죽더라도 해당 액터를 파괴시키지 않고 숨기고 Tick을 비활성화한다.<br>
그 이유로는 탱크에 카메라가 달려있기에 탱크가 죽더라도 해당 뷰에서 마무리하는 것이 목적이기 때문이다.

```C++
void ATank::HandleDestruction()
{
    Super::HandleDestruction();
    // 탱크는 숨기고 틱을 비활성화 시킨다.
    // 이유는 탱크가 터지더라도 카메라뷰는 남겨서 그 종료시점에 카메라는 볼 수 있게 하고 싶기 때문이다.
    // 대신 틱을 비활성화 했기에 탱크는 보이지도 조작하지도 못한다
    SetActorHiddenInGame(true);
    SetActorTickEnabled(false);
}
```

#### Tower

타워는 그냥 Destroy() 한다.

```C++
void ATower::HandleDestruction()
{
    Super::HandleDestruction();
    Destroy();
}
```

#### ToonTanksGameMode

게임 모드에서는 처리할 액터를 받으면 탱크냐 타워냐에 따라서 다르게 처리해준다.

```C++
void AToonTanksGameMode::ActorDied(AActor *DeadActor)
{
    if (DeadActor == Tank)
    {
        Tank->HandleDestruction();
        // 더이상 입력 못하게 하고 마우스 커서도 숨긴다.
        if (Tank->GetTankPlayerController())
        {
            Tank->DisableInput(Tank->GetTankPlayerController());
            Tank->GetTankPlayerController()->bShowMouseCursor = false;
        }
    }
    else if (ATower *DestroyedTower = Cast<ATower>(DeadActor))
    {
        DestroyedTower->HandleDestruction();
    }
}
```

#### HealthComponent

마지막으로 헬스 컴포넌트에서는 체력이 0 이하가 된다면 게임모드에 ActorDied()를 호출해서 체력이 0이하가 된 액터를 넘겨준다.

```C++

void UHealthComponent::DamageTaken(AActor *DamagedActor, float Damage, const UDamageType *DamageType, AController *Instigator, AActor *DamageCauser)
{
	if (Damage <= 0.f)
		return;

	Health -= Damage;

	if (Health <= 0.f && ToonTanksGameMode)
	{
		ToonTanksGameMode->ActorDied(DamagedActor);
	}
}
```

### 결과 실행

최종적으로 타워의 체력을 0을 만들면 아래와 같이 타워가 사라진다.

![13](/Assets/Images/Unreal/실습/ToonTanks/13.png)

또 플레이어 탱크의 체력이 0이 되면 더 이상 조작이 불가하고 탱크가 사라지지만 카메라는 그 위치에서 고정이 된다.

![14](/Assets/Images/Unreal/실습/ToonTanks/14.png)
