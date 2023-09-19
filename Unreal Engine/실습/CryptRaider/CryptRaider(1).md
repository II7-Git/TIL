# CryptRaider 1

## 기본 세팅 및 Asset

프로젝트는 아래 사진과 같이 1인칭 프로젝트를 통해 제작한다.<br>
이는 캐릭터 조작을 1인칭 기반으로 할 것이기 때문이다.

![1](/Assets/Images/Unreal/실습/CryptRaider/1.png)

사용할 에셋은 마켓 플레이스에 존재하는 무료 에셋인 Medival Dungeon을 사용한다

![2](/Assets/Images/Unreal/실습/CryptRaider/2.png)

## 레벨 구성

레벨은 아래와 같이 구성을 한다

![3](/Assets/Images/Unreal/실습/CryptRaider/3.png)

인 게임으로 보면 이러한 느낌의 던전을 구성하게 된다

![4](/Assets/Images/Unreal/실습/CryptRaider/4.png)

해당 내용을 정리할 때 사용된 중요한 내용들은 아래 링크에 정리해두었다.

[레벨 구성 시 TIP](./../../이론%20및%20정리/레벨%20구성%20시%20TIP.md)

## 라이트 구성

라이트 기능에 대한 자세한 정리는 아래 글에 해놓았다.

[라이트(Light) 기능 정리](<../../이론%20및%20정리/라이트(Light)%20정리.md>)

이를 바탕으로 환경 조명을 구성한 후 Torch를 수정하여 맵에 잘 배치해주어서 빛이 새지 않는 외벽 구조를 만든 후에 안에 조명 구성을 완료했다.

![4](/Assets/Images/Unreal/실습/CryptRaider/5.png)
![4](/Assets/Images/Unreal/실습/CryptRaider/6.png)
![4](/Assets/Images/Unreal/실습/CryptRaider/7.png)

## Mover 벽 생성

암호를 해결하면 문이 열리는 어드벤쳐 게임 형식이기에 암호가 풀렸을 때 열리는 조건을 가진 문이 필요하다. 그래서 Mover라는 클래스를 만들어 벽을 이동시키는 것을 구현했는데 이때 유용하게 쓰일 함수가 있다. 그 함수에 공식 문서를 참조한다.

[FMath::VInterpConstantTo](https://docs.unrealengine.com/5.2/en-US/API/Runtime/Core/Math/FMath/VInterpConstantTo/)

해당 함수의 원형을 보면 current위치를 target위치까지 일정한 스피드로 이동시키는 계산을 하여 FVector를 반환하는 함수인데 이를 잘 사용하면 쉽게 벽 이동을 구현할 수 있다.

구현 예시

```C++
if (ShouldMove)
	{
		FVector CurrentLocation = GetOwner()->GetActorLocation();
		FVector TargetLocation = OriginalLocation + MoveOffset;
		float Speed = FVector::Distance(OriginalLocation, TargetLocation) / MoveTime;

		FVector NewLocation = FMath::VInterpConstantTo(CurrentLocation, TargetLocation, DeltaTime, Speed);
		GetOwner()->SetActorLocation(NewLocation);
	}
```

위 처럼 Speed를 적절히 구해주고 목표 위치와 현재 위치를 계산 후 새로운 로케이션을 계산해 그 위치로 현재 액터를 변경해주면 원하는 기능을 구현할 수 있다.

![8](/Assets/Images/Unreal/실습/CryptRaider/8.png)
![9](/Assets/Images/Unreal/실습/CryptRaider/9.png)

## Grabber & 라인트레이스 기능 구현

현재 내가 보고있는 물체를 잡아서 들어올리는 기능의 구현하려 한다.<br>
그러기 위해서는 카메라가 보고 있는 물체가 무엇인지 파악해야 하는데 그때 사용하는 기능이 Line Trace 기능이다. 해당 기능에 대한 정리는 아래 글로 해놓았다.

[트레이스 기능 정리](/Unreal%20Engine/이론%20및%20정리/트레이스%20기능%20정리.md)

이 기능을 활용해서 현재 내가 어떠한 물체를 보고 있는지 정보를 알아내는 구현을 Grabber 클래스에 구현해내었다.

### DrawDebugSphere()

트레이스로 물체를 찾았을 때 물체의 좌표를 제대로 알아내는 것이 중요합니다. 이를 확인하기 위해서 DrawDebugSphere() 함수를 사용해서 Grab버튼을 눌렀을 때 걸린 벡터의 위치에 sphere를 그려서 대략적인 위치를 가늠해봅니다.

```C++
bool HasHit = GetWorld()->SweepSingleByChannel(
		HitResult,
		Start,
		End,
		FQuat::Identity,
		ECC_GameTraceChannel2,
		Sphere);

	if (HasHit)
	{
		DrawDebugSphere(GetWorld(), HitResult.Location, 10, 10, FColor::Green, false, 5);
		DrawDebugSphere(GetWorld(), HitResult.ImpactPoint, 10, 10, FColor::Red, false, 5);
	}
```

위의 코드처럼 HitResult에서 얻게되는 두 위치에 DrawDebugSphere()를 활용해봅니다.

아래 사진에서 초록구와 빨간구 위치를 보면 저희가 원하는 위치의 벡터를 구하면 Red의 위치인 HitResult.ImpactPoint가 필요한 벡터라는 것을 알 수 있습니다.
![10](/Assets/Images/Unreal/실습/CryptRaider/10.png)

## 물체 잡기 구현

### PhysicsHandleComponent

물체를 들어올리게 되면 원하는 위치로 물체를 들고다녀야하는데 이때 들고다니는 물체를 잡고 들고다니고 원하는 위치에 놓을때까지 벽을 통과하는 등의 물리적인 오류를 일으키면 안됩니다. 그렇기에 이때 사용되는 클래스가 UPhysicsHandleComponent 입니다.

[UPhysicsHandleComponent 공식 문서](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/PhysicsEngine/UPhysicsHandleComponent/)

이제 트레이스를 통해 원하는 액터를 얻은 뒤 PhysicsHandleComponent에 GrabComponentAtLocationWithRotation() 함수를 통해 세팅을 해줍니다.

Grab()과 Tick() 으로 잡고 이동하는 코드

```C++
//PhysicsHandle에서 잡은 물체를 플레이어 이동에 맞춰 따라 이동하게 세팅해주는 Tick함수
void UGrabber::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction *ThisTickFunction)
{
	Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
	UPhysicsHandleComponent *PhysicsHandle = GetPhysicsHandle();
	if (PhysicsHandle == nullptr)
	{
		return;
	}

	FVector TargetLocation = GetComponentLocation() + GetForwardVector() * HoldDistance;
	PhysicsHandle->SetTargetLocationAndRotation(TargetLocation, GetComponentRotation());
}

void UGrabber::Grab()
{
	UPhysicsHandleComponent *PhysicsHandle = GetPhysicsHandle();
	if (PhysicsHandle == nullptr)
	{
		return;
	}

	FVector Start = GetComponentLocation();
	FVector End = Start + GetForwardVector() * MaxGrabDistance;
	DrawDebugLine(GetWorld(), Start, End, FColor::Red);
	DrawDebugSphere(GetWorld(), End, 10, 10, FColor::Blue, false, 5);

	FCollisionShape Sphere = FCollisionShape::MakeSphere(GrabRadius);
	FHitResult HitResult;
	// FQuat: 회전값을 알려주는 객체 ,본문에 쓰인 FQuat::Identity는 회전값이 없다는 표현
	// SweepSingleByChannel():지정한 Collision Shape를 트레이스 시켜서 매개변수로 넣은 채널 소속의 첫번째 충돌체를 리턴하는 함수,충돌했다면 true리턴
	bool HasHit = GetWorld()->SweepSingleByChannel(
		HitResult,
		Start,
		End,
		FQuat::Identity,
		ECC_GameTraceChannel2,
		Sphere);

	if (HasHit)
	{
		//잡은 물체를 PhysicsHandle에 세팅해주기
		PhysicsHandle->GrabComponentAtLocationWithRotation(
			HitResult.GetComponent(),
			NAME_None,
			HitResult.ImpactPoint,
			GetComponentRotation());
	}
}

```

실제 구현한 모습입니다.
![11](/Assets/Images/Unreal/실습/CryptRaider/11.png)

PhysicsHandle을 사용했기에 아래 사진처럼 벽문에 걸리면 물리적인 오류없이 부드럽게 이동하는 모습을 확인할 수 있습니다.

![12](/Assets/Images/Unreal/실습/CryptRaider/12.png)

## 비밀 문 구현

### 콜리전 Overlap 이벤트 구현

문에 필요한 물건을 올려 두었을 때 문이 열리는 구조를 동작시키기 위해서는 문의 콜리전을 설치하고 해당 콜리전에 다른 액터가 들어왔을 때 이벤트가 발생해야합니다. 그렇기에 콜리전 반응을 Overlap으로 하고 Overlap이벤트를 켜주어야 합니다.

이 콜리전 반응에 대한 자세한 정리는 아래 글에 해놓았습니다.

[콜리전 정리](/Unreal%20Engine/이론%20및%20정리/콜리전%20정리.md)

### 액터 태그 설정

콜리전 내부의 플레이어를 비롯한 여러 액터들이 들어오게 될테지만 문을 열 수 있는 액터의 종류는 한정되야 합니다. 그래서 이러한 액터들에게 액터 태그를 달아주고 Overlap 이벤트 발동시 들어온 액터의 액터 태그를 조사하여 원하는 태그이면 동작하게 해줍니다.

![12](/Assets/Images/Unreal/실습/CryptRaider/13.png)

위 사진처럼 키 물건에 액터 태그를 달아주고 아래 코드처럼 액터의 태그를 비교해봅니다.

```C++
void UTriggerComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction *ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    TArray<AActor *> Actors;
    GetOverlappingActors(Actors);

    for (AActor *Actor : Actors)
    {
		//액터 태그 비교
        if (Actor->ActorHasTag(AcceptableActorTag))
        {
        }
    }
}
```
