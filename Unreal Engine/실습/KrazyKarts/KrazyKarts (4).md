# KrazyKarts (4)

## 클래스 컴포넌트로 리팩토링 하기

기존의 GoKart의 크기가 너무 커져서 움직임을 처리하는 부분과 서버 통신하는 Replication 부분을 액터 컴포넌트화 시켜서 분리하고 종속성이 있는 부분을 새롭게 수정하여 종속성을 끊어줘서 독자적으로 관리할 수 있게 구성했다.

### GoKart

먼저 GoKart는 기존에 움직임과 레플리케이션 담당하는 부분을 아래와 같이 <code>UGoKartMovementComponent</code>, <code>UGoKartMovementReplicator</code>를 만들어서 이곳에서 관리하게 구성했다.

```C++
UPROPERTY(VisibleAnywhere)
UGoKartMovementComponent *MovementComponent;

UPROPERTY(VisibleAnywhere)
UGoKartMovementReplicator *MovementReplicator;
```

### UGoKartMovementComponent

이 곳에선 움직임 처리에 관한 구성을 전부 리팩토링해서 구성해 넣었다. 기존의 움직임 Tick을 이곳에서 움직이게 하여 움직임은 이곳에서 총괄하게 됐다.

```C++
// Called every frame
void UGoKartMovementComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction *ThisTickFunction)
{
	Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

	if (GetOwnerRole() == ROLE_AutonomousProxy || GetOwnerRole() == ROLE_Authority)
	{
		LastMove = CreateMove(DeltaTime);
		SimulateMove(LastMove);
	}
}
```

### UGoKartMovementReplicator

이 곳은 레플리케이션의 관련된 기능을 구성하면서 리팩토링을 좀 했다. Tick을 보면 서버 쪽에서 문제 현상을 해결하기 위해 UpdateServerState를 통해서 서버 스테이트만 업데이트하고 이 정보를 가지고 MovementComponent에서 알아서 움직임을 처리하게 하는 구조로 바꾸었다.

```C++
// Called every frame
void UGoKartMovementReplicator::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction *ThisTickFunction)
{
	Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

	if (MovementComponent == nullptr)
		return;

	FGoKartMove LastMove = MovementComponent->GetLastMove();
	if (GetOwnerRole() == ROLE_AutonomousProxy)
	{
		UnacknowledgedMoves.Add(LastMove);
		Server_SendMove(LastMove);
	}

	// 서버이면서 동시에 특정 Pawn을 조종하고 있는 경우
	// if (GetOwner()->GetRemoteRole() == ROLE_SimulatedProxy)
	if (GetOwnerRole() == ROLE_Authority)
	{
		UpdateServerState(LastMove);
	}
	/*
	각 클라이언트에서 조종하고 있지 않은 폰들이 OnRep_ServerState로 업데이트된
	정보에 따라서 예측해서 동작을 취하여 자연스럽게 움직이게 하기
	*/
	if (GetOwnerRole() == ROLE_SimulatedProxy)
	{
		MovementComponent->SimulateMove(ServerState.LastMove);
	}
}

```

```C++
void UGoKartMovementReplicator::UpdateServerState(const FGoKartMove &Move)
{
	ServerState.LastMove = Move;
	ServerState.Transform = GetOwner()->GetActorTransform();
	ServerState.Velocity = MovementComponent->GetVelocity();
}
```

---

이렇게 컴포넌트를 통한 리팩토링을 통해서 클래스 관리 유지가 편해지게 변화시켰다.
