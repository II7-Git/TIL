# SimpleShooter (2)

## 애니메이션 블루프린트

애니메이션의 로직 구현은 유니티와 마찬가지로 이 곳에서도 C++로 하기에 적합하지 않다. 그래서 언리얼에서도 결과를 보면서 작업하기 유용한 애니메이션 블루프린트를 생성하여 애니메이션의 로직을 구성하게 된다.

처음 애니메이션 블루프린트를 만들면 볼 수 있는 그래프인데 저 출력 포즈칸에 최종적인 로직의 결과물을 넣게 됨으로써 애니메이션의 종합적인 구조를 출력하게 되는 방식이다.
![3](/Assets/Images/Unreal/실습/SimpleShooter/3.png)

### 보행 이동 애니메이션 - 블렌드 스페이스

이동에 관한 애니메이션은 360도 방위와 속도에 따른 애니메이션 블렌딩을 해야하기에 기존에 방식들로는 원활한 블렌딩의 어려움을 겪을 수 있다. 이럴 때 사용하면 좋은 언리얼의 기능으로 블렌드 스페이스가 있다.

블렌드 스페이스는 2차원 구조에서 변수에 따라서 애니메이션을 블렌드해주는 기능을 가지고 있는데 보행 이동의 구현에서는 예를 들면 Angle과 Speed 변수를 만들고 해당 각도와 스피드에 따른 애니메이션들을 넣으면 된다. <br>
예를 들어 스피드 50의 앵글이 90도면 오른쪽으로 걷는 애니메이션을 넣어주고 스피드 0 앵글 0 이면 가만히 서있는 동작을 넣어준다.<br>
이렇게 보행 이동 애니메이션같이 여러 애니메이션이 수치에 따라 섞일 때는 블렌드 스페이스 기능이 유용하다.

아래 사진은 실제 Angle과 Speed를 각 축으로 하여 애니메이션을 블렌드 스페이스에 배치한 모습이다.
![4](/Assets/Images/Unreal/실습/SimpleShooter/4.png)

### 이동 애니메이션 블루프린트

아래는 애니메이션 블루프린트에서 이동에 대한 애니메이션 로직이다.
![5](/Assets/Images/Unreal/실습/SimpleShooter/5.png)

여기서 우리가 설정할 값은 Speed와 Angle인데 이 값은 애니메이션을 취할 Pawn에게서 얻게 된다.

Speed: 스피드는 Pawn의 Velocity(속도)의 값을 얻어와서 세팅해준다.

Angle: 현재 플레이어의 회전 각도는 플레이어 기준에서의 로컬 좌표에서 어떻게 회전하고 있는지를 얻어와야한다.<br>
이 때 벡터의 역방향 트랜스폼을 구해야 하는데 그 이유는 Pawn에게서 Velocity를 얻을 때 이 값은 월드 좌표계를 기준으로 하고 있다.<br>
하지만 우리는 로컬 좌표에서 회전을 얼마나 하고 있는지 알고 싶은 것이므로 이 Velocity를 현재 Pawn 기준으로 역방향 트랜스폼을 하여 값을 얻어낼 필요가 있다.<br>
그래서 위처럼 Inverse Transform을 하여 Velocity를 로컬 좌표로 변경하여 Yaw 회전값을 얻어내야 올바르게 캐릭터의 이동을 처리할 수 있다.

### 애니메이션 속도와 일치시키기

이동에 관한 속도에 따른 애니메이션 속도를 처리하기 위해서 각 애니메이션에 들어가서 한 동작당 이동하는 속도를 알아낸 뒤 이를 이동 블렌드 스페이스의 속도 값과 일치시켜 주었다.

최종적으로 미끄러지는 동작이 아닌 자연스럽게 이동하는 모습을 구현해낼 수 있었다.
![6](/Assets/Images/Unreal/실습/SimpleShooter/6.png)

## Gun Actor 제작

슈팅 게임에서 플레이어는 총기를 자유롭게 변경하고 사용하는 것이 중요하기에 이를 위해 Gun Actor를 만들고 이를 플레이어의 손에 붙이는 작업이 필요하다. 이를 과정으로 적어보면

1. Gun Actor 생성
2. Gun Actor 스폰
3. 손 위치에 Gun Actor 부착

정도의 과정이 될 것이다. 이를 하나씩 구현해보았다.

### Gun Actor 생성

Gun을 Actor를 상속받아서 만든다. 씬에 배치될 것이기에 USceneComponent를 만들어 RootComponent로 만들어주고 에셋에서 제공되는 총기의 메시 종류에 따라서 Mesh를 만들어주고 RootComponent에 붙혀준다.

```C++
AGun::AGun()
{
	// Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
	PrimaryActorTick.bCanEverTick = true;

	Root = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));
	SetRootComponent(Root);

	Mesh = CreateDefaultSubobject<USkeletalMeshComponent>(TEXT("Mesh"));
	Mesh->SetupAttachment(Root);
}
```

이렇게 해서 만든 클래스를 상속 받은 블루클래스를 만들고 그곳에 원하는 총기 메시를 붙혀준다.
![11](/Assets/Images/Unreal/실습/SimpleShooter/11.png)

### GunActor 스폰 후 손에 붙히기

주의할 점은 현재 캐릭터 메시에서 기본적으로 제공되는 무기가 있기에 해당 무기를 숨겨주어야한다. 그리고 그 위치에 Actor를 붙히기 위해서 소켓을 하나 만들어 주어야한다.

먼저 메시를 열어서 무기가 어느 메시 밑에 붙어있는지 확인해야한다. 확인해보면 "weapon_r" 밑에 붙어있는 것을 확인할 수 있는데 이러면 이 "weapon_r"을 숨겨준다.<br>
또 무기를 붙힐 소켓을 "WeaponSocket"이라는 이름으로 "weapon_r" 밑에 만들어준다.
![11](/Assets/Images/Unreal/실습/SimpleShooter/7.png)

그 뒤 플레이어의 메시를 가져와서 다음 이름의 메시를 숨기는 코드이다.

```C++
GetMesh()->HideBoneByName(TEXT("weapon_r"), EPhysBodyOp::PBO_None);
```

다음은 총기 액터를 만들어준 뒤 우리가 만든 소켓에 메시를 붙히는 작업이다. 다음 코드에서 AttachToComponent()를 이용해 플레이어 메시에서 "WeaponSocket"을 찾아서 그 곳에 Gun Actor를 붙혀준다.<br> SetOwner()를 해준 이유는 전투를 처리할 때 총기를 발사한 주체가 누구인지 알아야하기 때문이다.

```C++
    Gun = GetWorld()->SpawnActor<AGun>(GunClass);
	Gun->AttachToComponent(GetMesh(), FAttachmentTransformRules::KeepRelativeTransform, TEXT("WeaponSocket"));
    //총기의 주인 설정
	Gun->SetOwner(this);
```

이를 완성하고 실행하면 총기가 손 쪽에 있지만 위치가 살짝 맞지 않는 것을 확인할 수 있는데 이는 실행하고 메시를 이동시켜서 손에 딱 맞는 위치로 조정해준다.

![8](/Assets/Images/Unreal/실습/SimpleShooter/8.png)

그 뒤 현재 총의 트랜스폼을 복사해주고 블루프린트로 가서 그 위치를 붙혀넣기 해준다.
![9](/Assets/Images/Unreal/실습/SimpleShooter/9.png)

이러면 총의 위치가 손에 위치에 맞게 배치된다.
![10](/Assets/Images/Unreal/실습/SimpleShooter/10.png)

## 총기 발사 구현

### 발사 함수 틀 구현

먼저 총기 발사를 처리하기 위한 발사 함수를 구현해야한다.

총기 입장에서는 플레이어나 적이 모두 발사를 누를 수 있기에 PullTrigger()라는 함수를 만들고 플레이어는 PullTrigger()를 실행시킬 함수를 만든다.

Gun.Cpp

```C++
void AGun::PullTrigger()
{
}
```

Player.Cpp 는 액션 바인딩을 해준다.

```C++
void AShooterCharacter::Shoot()
{
	Gun->PullTrigger();
}
```

### 발사 이펙트 설정

발사 시 총구에서 나오는 이펙트는 총구에 붙혀 있어야하기에 SpawnEmitterAttached()로 소켓에 붙혀야한다.

먼저 소켓의 이름을 알아낸다.
![12](/Assets/Images/Unreal/실습/SimpleShooter/12.png)

그 뒤 총기 발사 버튼이 눌리면 해당 소켓에 이펙트를 부착하여 발동시킨다.

```C++
void AGun::PullTrigger()
{
	UGameplayStatics::SpawnEmitterAttached(MuzzleFlash, Mesh, TEXT("MuzzleFlashSocket"));
}
```

![13](/Assets/Images/Unreal/실습/SimpleShooter/13.png)

### RayCasting을 통한 발사 위치 설정

TPS에서 에임의 기준은 플레이어의 카메라에서 보이는 크로스헤어가 기준이 된다.<br>
즉 카메라에서 보는 위치의 벡터를 향해 총이 발사되어야 한다는 뜻이고 이는 카메라에서 보는 방향으로 RayCasting을 하여 해당 위치를 알아내야 적합하다는 뜻이다.

그래서 먼저 카메라의 위치를 알아내기 위해서 DrawDebugCamera()를 통해 카메라 위치를 알아낸다.

```C++
void AGun::PullTrigger()
{
	UGameplayStatics::SpawnEmitterAttached(MuzzleFlash, Mesh, TEXT("MuzzleFlashSocket"));

	APawn *OwnerPawn = Cast<APawn>(GetOwner());
	if (OwnerPawn == nullptr)
		return;
	AController *OwnerController = OwnerPawn->GetController();
	if (OwnerController == nullptr)
		return;

	FVector Location;
	FRotator Rotation;
	OwnerController->GetPlayerViewPoint(Location, Rotation);

	DrawDebugCamera(GetWorld(), Location, Rotation, 90, 2, FColor::Red, true);
}

```

디버그 시 카메라가 플레이어 카메라 위치에 그려진 모습
![14](/Assets/Images/Unreal/실습/SimpleShooter/14.png)

### LineTraceSingleByChannel()

다음은 현재 구한 카메라 위치에서 LineTrace를 하여 원하는 채널에 속한 액터와 충돌할 시 값을 얻어내는 LineTrace를 실행한다.

아래 코드는 카메라의 위치에서 목표로 한 End위치를 구해서 카메라 위치부터 End까지 라인 트레이스를 하여 내가 설정한 Bullet 트레이스 채널에 대해서 충돌한 HitResult를 리턴 받아서 그 위치에 디버그 드로우를 하는 코드이다.

```C++
void AGun::PullTrigger()
{
	UGameplayStatics::SpawnEmitterAttached(MuzzleFlash, Mesh, TEXT("MuzzleFlashSocket"));

	APawn *OwnerPawn = Cast<APawn>(GetOwner());
	if (OwnerPawn == nullptr)
		return;
	AController *OwnerController = OwnerPawn->GetController();
	if (OwnerController == nullptr)
		return;

	FVector Location;
	FRotator Rotation;
	OwnerController->GetPlayerViewPoint(Location, Rotation);

	// 현재 카메라 위치에서 카메라가 보는 방향으로 MaxRange만큼 이동한 벡터
	FVector End = Location + Rotation.Vector() * MaxRange;

	// TODO : LineTrace //ECC_GameTraceChannel1:만든 트레이스 채널 Bullet의 이름
	FHitResult Hit;
	bool bSuccess = GetWorld()->LineTraceSingleByChannel(Hit, Location, End, ECollisionChannel::ECC_GameTraceChannel1);
	if (bSuccess)
	{
		DrawDebugPoint(GetWorld(), Hit.Location, 20, FColor::Red, true);
	}
}
```

이를 통해 실행해보면 아래와 같이 총을 쐈을 때 MaxRange라는 사거리 안에서 충돌한 위치를 리턴하는 것을 확인할 수 있다.
![15](/Assets/Images/Unreal/실습/SimpleShooter/15.png)
