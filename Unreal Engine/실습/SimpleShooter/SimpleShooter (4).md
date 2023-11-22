# SimpleShooter (4)

## 적 AI 만들기

이번 적 AI는 AIController의 기능을 이용해서 구현을 해본다.그래서 AIController를 기반으로한 클래스와 파생 블루프린트까지 만들어준다.

## 플레이어 포커스

AIController에 내장된 기능 중에 포커스 기능을 사용해 플레이어를 향하도록 설정한다.

### SetFocalPoint()

AIController에 내장된 기능으로 특정 벡터 위치를 향하도록 하는 메소드로 주로 고정된 액터의 위치에 사용한다.

### SetFocus()

특정 액터를 따라가면서 포커스를 할 때 사용하는 기능으로 움직이는 액터를 포커스 하려 할 때 사용한다.

### ClearFocus()

설정된 포커스를 해제시킨다.

---

이제 SetFocus()를 통해 플레이어를 추적할 수 있게끔 구현한다.

## 플레이어에게 접근하기

### NavMesh를 사용한 이동

AIController의 NavMesh를 사용한 경로 생성을 통해 플레이어를 향해서 장애물들을 피해서 이동하도록 구성한다.

NavMesh를 설정해주기 위해서는 먼저 Mesh를 계산하기 위해서 NavMesh Bound Volume을 설정해주어야한다.
![20](/Assets/Images/Unreal/실습/SimpleShooter/20.png)

이 박스를 배치하면 아래와 같이 박스 안에 요소들에서 걸을 수 있는 경로를 계산해준다.
![22](/Assets/Images/Unreal/실습/SimpleShooter/22.png)

### MoveToActor()

### MoveToLocation()

포커스처럼 특정 액터 혹은 벡터를 향해 FMesh를 따라 경로를 생성하여 이동하게끔하는 메소드들이다.

현재까지의 기능들을 코드로 AIController에 구현하면

```C++
void AShooterAIController::BeginPlay()
{
    Super::BeginPlay();
    PlayerPawn = UGameplayStatics::GetPlayerPawn(GetWorld(), 0);
    SetFocus(PlayerPawn);
}

void AShooterAIController::Tick(float DeltaSeconds)
{
    Super::Tick();
    //(어떤 액터를 따라갈 것인지, 액터와의 거리(2m))
    MoveToActor(PlayerPawn, 200);
}

```

이를 실행하면 아래 사진처럼 플레이어를 포커스한채로 2M 간격으로 계속 따라오는 적 AI를 확인할 수 있다.
![23](/Assets/Images/Unreal/실습/SimpleShooter/23.png)
![24](/Assets/Images/Unreal/실습/SimpleShooter/24.png)

## 적 AI 시야 제한하기

플레이어가 보이지 않는 상황에서 적 AI는 플레이어를 알아채고 쫒아와서는 안된다. 그래서 적의 시야에 들지 않거나 벗어나면 쫒지 않게 하는 기능을 구현해본다.

### LineOfSightTo()

AIController에 내장된 함수로 시야에 있는지 여부를 bool 값으로 리턴해준다. 이를 이용해 시야에 들어왔을 시에만 Focus를 하고 따라오게 설정해준다.

```C++
void AShooterAIController::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);

    if (LineOfSightTo(PlayerPawn))
    {
        SetFocus(PlayerPawn);
        //(어떤 액터를 따라갈 것인지, 액터와의 거리)
        MoveToActor(PlayerPawn, AcceptanceRadius);
    }
    else
    {
        // 해당 우선순위까지의 포커스 해제 0:Default,1:Move,2:GamePlay
        ClearFocus(EAIFocusPriority::Gameplay);
        StopMovement();
    }
}
```

이를 이용하면 아래 사진처럼 시야에서 벗어나면 쫒아오지 않는 적을 볼 수 있다
![25](/Assets/Images/Unreal/실습/SimpleShooter/25.png)

## 행동 트리와 블랙보드

이렇게 코드로 AI를 구성할 수 있지만 좀 더 복잡한 AI를 구성하기 위해서는 행동 트리와 블랙보드를 활용하면 유용하다.

### 행동 트리

행동 트리는 여러 행동에 따른 알고리즘을 트리화해서 배치하여 좀 더 편하게 AI를 구현할 수 있다. 이를 통해서 자연스러운 움직임을 구현할 수 있다.

### 블랙보드

블랙보드는 쉽게 말해 AI의 메모리 기능을 담당한다. 코드에서 구현한 기능이라던지 변수 등 게임상의 프로퍼티를 블랙보드화해서 위의 행동트리에서 사용하여 어떠한 행동을 할지를 바꿀 수 있게하는 역할을 한다.

### 실제 사용

AIController에서는 만든 행동 트리를 가져와서 RunBehaviorTree()로 행동 트리를 작동시킨다.

그 뒤 블랙보드에 생성한 변수의 이름을 Key값으로 하여 원하는 변수를 C++에서 설정이 가능하다.

```C++
void AShooterAIController::BeginPlay()
{
    Super::BeginPlay();
    PlayerPawn = UGameplayStatics::GetPlayerPawn(GetWorld(), 0);

    if (AIBehavior != nullptr)
    {
        RunBehaviorTree(AIBehavior);

        GetBlackboardComponent()->SetValueAsVector(TEXT("PlayerLocation"), PlayerPawn->GetActorLocation());
        GetBlackboardComponent()->SetValueAsVector(TEXT("StartLocation"), GetPawn()->GetActorLocation());
    }
}

```

위에서 블랙보드의 있는 변수를 설정할 수 있고 이 변수를 사용하여 아래처럼 행동트리에서 AI를 구성할 수 있다.
![26](/Assets/Images/Unreal/실습/SimpleShooter/26.png)

### BTTask 와 BTService

#### BTTask

BTTask는 C++ 코드로 원하는 기능을 구현하여 이를 블랙보드에서 사용할 노드로 만들어서 사용하는 방식을 뜻한다.

구현부만 살펴보자면 아래처럼 노드 이름을 생성자에서 정해주고 ExecuteTask()나 TickTask() 등의 BTTask의 메소드를 상속받아서 실행할 동작을 구현하면 된다.

BTTask_Shoot.cpp

```C++
UBTTask_Shoot::UBTTask_Shoot()
{
    NodeName = TEXT("Shoot");
}

EBTNodeResult::Type UBTTask_Shoot::ExecuteTask(UBehaviorTreeComponent &OwnerComp, uint8 *NodeMemory)
{
    Super::ExecuteTask(OwnerComp, NodeMemory);

    if (OwnerComp.GetAIOwner() == nullptr)
    {
        return EBTNodeResult::Failed;
    }
    AShooterCharacter *Character = Cast<AShooterCharacter>(OwnerComp.GetAIOwner()->GetPawn());

    if (Character == nullptr)
    {
        return EBTNodeResult::Failed;
    }

    Character->Shoot();
    return EBTNodeResult::Succeeded;
}
```

이를 블랙보드에서 활용하면 위에 코드에서 지은 노드네임을 바탕으로 노드를 배치하고 사용할 수 있는 모습을 볼 수 있다.

![27](/Assets/Images/Unreal/실습/SimpleShooter/27.png)

#### BTService

BTService는 노드의 부착되어 사용되며 해당 노드가 실행되는 동안 반복해서 발동되는 기능을 뜻하며 이 BTService를 통해서 이를 C++로 구현할 수 있다.

BTService의 예로 해당 서비스가 실행된 노드의 정보를 가져와서 활용하는 모습을 볼 수 있다.

BTService_PlayerLocation.cpp

```C++

#include "BTService_PlayerLocation.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "Kismet/GameplayStatics.h"
#include "GameFrameWork/Pawn.h"

UBTService_PlayerLocation::UBTService_PlayerLocation()
{
    NodeName = "Update Player Location";
}

void UBTService_PlayerLocation::TickNode(UBehaviorTreeComponent &OwnerComp, uint8 *NodeMemory, float DeltaSeconds)
{
    Super::TickNode(OwnerComp, NodeMemory, DeltaSeconds);

    APawn *PlayerPawn = UGameplayStatics::GetPlayerPawn(GetWorld(), 0);
    if (PlayerPawn == nullptr)
    {
        return;
    }

    OwnerComp.GetBlackboardComponent()->SetValueAsVector(GetSelectedBlackboardKey(), PlayerPawn->GetActorLocation());
}
```

구현이 끝났으면 노드에 실제 부착해서 사용한다.

![28](/Assets/Images/Unreal/실습/SimpleShooter/28.png)

### 최종 완성된 블랙 보드

![29](/Assets/Images/Unreal/실습/SimpleShooter/29.png)

## 죽음에 따른 처리

플레이어나 적이 죽어도 총이 계속 발사되는 현상을 볼 수 있는데 이를 막기 위해 체력이 0이 되면 Controller를 제거하고 Collision을 꺼주는 작업을 진행했다. 이를 통해서 시체만 남는 것을 확인할 수 있다.

```C++
float AShooterCharacter::TakeDamage(float DamageAmount, struct FDamageEvent const &DamageEvent, class AController *EventInstigator, AActor *DamageCauser)
{
	float DamageToApply = Super::TakeDamage(DamageAmount, DamageEvent, EventInstigator, DamageCauser);
	DamageToApply = FMath::Min(Health, DamageToApply);
	Health -= DamageToApply;

	// 죽으면 Controller제거하고 Collision도 제거
	if (IsDead())
	{
		DetachFromControllerPendingDestroy();
		GetCapsuleComponent()->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	}

	return DamageToApply;
}
```
