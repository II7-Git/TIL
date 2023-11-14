# SimpleShooter (3)

## 데미지 처리

### 총기의 TakeDamage() 실행

데미지 처리는 TakeDamage()를 사용해서 구현했다. 이전에 구현했던 라인트레이스로 충돌한 물체가 존재할 시 해당 물체에 대해서 TakeDamage()를 발동시킨다.

```C++
// 데미지 처리
		AActor *HitActor = Hit.GetActor();
		if (HitActor)
		{
			FPointDamageEvent DamageEvent(Damage, Hit, ShotDirection, nullptr);
			HitActor->TakeDamage(Damage, DamageEvent, OwnerController, this);
		}
```

### 캐릭터의 데미지 처리

위의 HitActor가 플레이어일 시 체력이 깎이는 처리를 해주어야한다. 그러기 위해서 ShooterCharacter 파일의 체력을 추가해준다.

```C++
//ShooterCharacter.h에 체력 선언
UPROPERTY(EditDefaultsOnly)
float MaxHealth = 100;

UPROPERTY(VisibleAnywhere)
float Health;

//ShooterCharacter.cpp에서 체력 시작시에 초기화
void AShooterCharacter::BeginPlay()
{
	Super::BeginPlay();

	Health = MaxHealth;
}
```

체력을 위처럼 구현했다면 TakeDamage함수를 Override해주어야한다. 그 이유는 Actor의 TakeDamage()로는 체력을 줄일 수 없기 때문이다. 따라서 이를 Override해서 구현해준다. 이때 체력이 0밑으로 떨어지지 않기 위해 깎이는 양을 FMath::Min()으로 조절해준다.

```C++
float AShooterCharacter::TakeDamage(float DamageAmount, struct FDamageEvent const &DamageEvent, class AController *EventInstigator, AActor *DamageCauser)
{
	float DamageToApply = Super::TakeDamage(DamageAmount, DamageEvent, EventInstigator, DamageCauser);
	DamageToApply = FMath::Min(Health, DamageToApply);
	Health -= DamageToApply;

	return DamageToApply;
}
```

이 체력을 로그로 띄워서 제대로 줄어드는지 확인해보았다.

![16](/Assets/Images/Unreal/실습/SimpleShooter/16.png)

## 사망 애니메이션 구현

먼저 애니메이션 블루프린트에서 IsDead변수를 만들어 이에 따라 사망 액션이 재생되게 블렌드 애니메이션을 수행했다.

![18](/Assets/Images/Unreal/실습/SimpleShooter/18.png)

이제 이 IsDead를 수정해야 하는데 이는 ShooterCharacter에서 HP가 0이 됐을 때에 따라서 진행해야한다.

아래 코드를 보면 UFUNCTION(BlueprintPure)를 쓴 것을 확인할 수 있는데 이렇게 하면 퓨어 노드가 되서 블루프린트에서 값을 가져오기만 가능하고 수정은 불가하게 된다. 대신 처리가 빠르기에 값을 가져오려고 할 때 사용하면 유용하다. <br>
또 이는 C++에서 Const 기능과 유사하기에 C++에서도 Const와 같이 쓰는 경우가 많다.

```C++
// ShooterCharacter.h
public:
	// 퓨어 노드는 항상 같은 값만 반환하고 호출한다해도 아무것도 바꾸지 않기에 계산이 빠르다
	// 그래서 const와 함께 많이 쓴다
	UFUNCTION(BlueprintPure)
	bool IsDead() const;

// ShooterCharacter.cpp
bool AShooterCharacter::IsDead() const
{
	return Health <= 0;
}

```

아래 사진을 보면 IsDead에는 실행 핀이 없는 것을 확인할 수 없다. 이게 퓨어노드의 형태이다.<br>
그리고 이 값을 이용해 애니메이션 블루프린트에 있는 IsDead변수를 변경해준다.
![19](/Assets/Images/Unreal/실습/SimpleShooter/19.png)

이를 완료해서 최종적인 실행을 해보면 체력이 0이 되면 죽는 캐릭터를 볼 수 있다.
![17](/Assets/Images/Unreal/실습/SimpleShooter/17.png)
