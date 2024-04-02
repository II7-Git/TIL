# PuzzlePlatform (1)

## 멀티 플레이 기본 개념 연습

CMD 창을 통한 서버, 클라이언트의 접속
![2](/Assets/Images/Unreal/실습/PuzzlePlatforms/2.png)

서버 콘솔창에서 접속 여부 확인과 플레이어 두명 접속된 것을 확인
![3](/Assets/Images/Unreal/실습/PuzzlePlatforms/3.png)

#### [Replication 과 Authority 정리](</Unreal%20Engine/이론%20및%20정리/멀티플레이/Replication(복제)와%20Authority(권한).md>)

```C++
void AMovingPlatform::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 자신이 서버인지 확인하는 메소드 , True면 서버
    if (HasAuthority())
    {
        FVector Location = GetActorLocation();
        Location += FVector(Speed * DeltaTime, 0, 0);

        SetActorLocation(Location);
    }
}

```

이를 동작시켜보면 아래처럼 서버에서만 움직이는 플랫폼을 볼 수 있는데 이러면 클라이언트는 없는 위치에 플랫폼이 존재해 서로 충돌 등의 문제가 생겨서 동기화 측면에서 문제를 확인
![4](/Assets/Images/Unreal/실습/PuzzlePlatforms/4.png)

### FVector 관련 처리

#### MakeEditWidget

FVector가 에디터상에서는 그래픽적으로 확인하기 어려워서 실제 어느 포지션에 있는지 모르는 경우가 많은데 이를 해결하기 위한 방법으로 MakeEditWidget이 있다.

```C++
UPROPERTY(EditAnywhere, Meta = (MakeEditWidget = true))
FVector TargetLocation;
```

위의 코드처럼 UPROPERTY 상에서 Meta = (MakeEditWidget = true) 옵션을 넣게 되면 해당 프로퍼티에 대한 위젯이 만들어져서 실체가 없는 정보도 에디터상에서 확인할 수 있게되는 기능으로 유용하게 사용할 수 있다.

#### 벡터의 글로벌화, 정규화

벡터의 글로벌화는 어떤 액터등에 귀속된 상대 벡터들이나 글로벌 벡터사이에 계산에서 잘못된 오류를 범할 수 있기에 그전에 글로벌화 시켜주는 것이 필요하다.

상대 벡터를 글로벌화 해주는 함수

```C++
FVector GlobalTargetLocation = GetTransform().TransformPosition(TargetLocation);
```

벡터를 정규화 해주는 함수는 여러개 있지만 벡터를 새로 생성해내지 않는 GetSafeNormal을 사용했다.

```C++
FVector Direction = (GlobalTargetLocation - Location).GetSafeNormal();
```

### 퍼즐 플랫폼 구성

퍼즐 플랫폼을 좀 더 게임의 요소를 활용할 수 있게 맵을 구성했다.

기존의 움직이는 플랫폼은 두 지점 사이를 왕복 이동하는 코드로 변경하여 특정 위치를 오가는 플랫폼화 시켰다.

```C++
void AMovingPlatform::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 자신이 서버인지 확인하는 메소드 , True면 서버
    if (HasAuthority())
    {
        FVector Location = GetActorLocation();

        // 이동해야할 최대 거리
        float JourneyLength = (GlobalTargetLocation - GlobalStartLocation).Size();
        // 현재까지 이동한 거리
        float JourneyTravelled = (Location - GlobalStartLocation).Size();

        // 이동 거리가 목표 지점을 넘어섰으면 되돌아가게 설정
        // 시작 벡터와 목표 벡터를 교환
        if (JourneyTravelled >= JourneyLength)
        {
            FVector Swap = GlobalStartLocation;
            GlobalStartLocation = GlobalTargetLocation;
            GlobalTargetLocation = Swap;
        }
        // 현재 위치에서 타겟 위치까지의 벡터
        FVector Direction = (GlobalTargetLocation - GlobalStartLocation).GetSafeNormal();
        Location += Speed * DeltaTime * Direction;

        SetActorLocation(Location);
    }
}
```

또 특정 플랫폼은 발판을 밟아야지만 움직이는 기능을 넣기 위해서 먼저 위에 충돌을 감지할 수 있는 트리거 공간을 가진 발판을 만들었다. 이것의 이름은 PlatformTrigger로 구성했다.

이를 통해서 일단 간단히 구현된 퍼즐 플랫폼의 형태는 아래와 같다.

![5](/Assets/Images/Unreal/실습/PuzzlePlatforms/5.png)

#### 발판 기능 보강

발판에 연결된 MovingPlatform을 관리하는 TArray를 활용해서 해당 발판을 밟으면 움직일 수 있게하고 나가면 움직이지 못하게 하는 구성을 했다.

OnComponentBeginOverlap, OnComponentEndOverlap 각각의 AddDynamic을 통해 함수를 이벤트로 등록해놓아서 밟고 나갈 때마다 해당 함수가 발동되게 구성됐다.

```C++
// Called when the game starts or when spawned
void APlatformTrigger::BeginPlay()
{
	Super::BeginPlay();

	if (TriggerVolume != nullptr)
	{
		TriggerVolume->OnComponentBeginOverlap.AddDynamic(this, &APlatformTrigger::OnOverlapBegin);
		TriggerVolume->OnComponentEndOverlap.AddDynamic(this, &APlatformTrigger::OnOverlapEnd);
	}
}

void APlatformTrigger::OnOverlapBegin(class UPrimitiveComponent *OverlappedComp, class AActor *OtherActor, class UPrimitiveComponent *OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult &SweepResult)
{
	for (AMovingPlatform *Platform : PlatformsTrigger)
	{
		Platform->AddActiveTrigger();
	}
}

void APlatformTrigger::OnOverlapEnd(class UPrimitiveComponent *OverlappedComp, class AActor *OtherActor, class UPrimitiveComponent *OtherComp, int32 OtherBodyIndex)
{
	for (AMovingPlatform *Platform : PlatformsTrigger)
	{
		Platform->RemoveActiveTrigger();
	}
}
```

#### GameInstance

[GameInstance 정리](/Unreal%20Engine/이론%20및%20정리/GameInstance.md)

### Travel을 통한 이동

#### ServerTravel

ServerTravel을 월드에서 실행하는 기능으로 현재 월드에 연결된 컨트롤러들의 연결을 끊기지 않고 임의의 링크로 지정한 다른 레벨로 전부 이동시키는 기능을 하는 메소드이다.

사용 예는 아래의 코드와 같으며 월드에서 실행하는 기능이므로 UWorld의 정보를 가져와서 실행하게 된다.

이떄 아래 ServerTravel을 보면 링크 끝에 '?listen'을 볼 수 있는데 이는 링크에 대한 옵션으로 서버 옵션에서 listen server로 레벨을 가동한다는 뜻으로 이를 통해서 서버의 역할로 실행하게 된다.

```C++

void UPuzzlePlatformsGameInstance::Host()
{
    UEngine *Engine = GetEngine();

    if (Engine == nullptr)
        return;

    Engine->AddOnScreenDebugMessage(0, 10, FColor::Green, TEXT("Hosting"));

    UWorld *World = GetWorld();

    if (World == nullptr)
        return;

    World->ServerTravel("/Game/ThirdPerson/Maps/ThirdPersonMap?listen");
}
```

#### ClientTravel

ClientTravel은 말 그대로 클라이언트가 실행하는 레벨 이동으로 각 Controller에서 실행해서 해당 Controller를 가진 Client만이 링크의 레벨로 이동하게 된다.

코드 예) (Address는 서버에 IP 주소가 들어간다)<br>
각 클라이언트에서 Controller를 얻어와서 Controller에서 Travel을 시도하는 것을 볼 수 있다.

```C++
void UPuzzlePlatformsGameInstance::Join(const FString &Address)
{
    UEngine *Engine = GetEngine();

    if (Engine == nullptr)
        return;

    Engine->AddOnScreenDebugMessage(0, 20, FColor::Green, FString::Printf(TEXT("Joining %s"), *Address));

    APlayerController *PlayerController = GetFirstLocalPlayerController();

    if (PlayerController == nullptr)
        return;

    PlayerController->ClientTravel(Address, ETravelType::TRAVEL_Absolute);
}
```

---

이를 통해서 서버에서는 호스팅 기능을 클라이언트는 해당 서버에 접속하는 기능을 수행하게 되고 그 결과를 아래 사진으로 확인했다.

![9](/Assets/Images/Unreal/실습/PuzzlePlatforms/9.png)
