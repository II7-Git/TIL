# KrazyKarts (6)

## 보간된 움직임과 Collider의 분리

기존 SimulatedProxy의 보간된 움직임은 사실 서버에서의 진짜 움직임과는 다른 움직임을 취한다. 그래서 만약 클라이언트에서의 SimulatedProxy 움직임의 중간에 충돌체가 있고 서버쪽에서는 없다면 SimulatedProxy만 충돌 이벤트가 생기는 문제가 발생할 수 도 있다.

이를 해결하기 위해서 Collider와 MeshOffset을 분리하여 MeshOffset은 보간된 움직임을 취하고 실제 Collider는 서버에서 받는 정보만 사용하는 방법으로 근사화를 해서 문제를 보완하는 방법이 있다.

구현하는 방법은 아래와 같다.

먼저 MeshOffset 과 box Collider를 분리해낸다.
![2](/Assets/Images/Unreal/실습/KrazyKarts/2.png)

다음 SimulatedProxy의 복사에서 보간된 움직임은 MeshOffset에서만 일어나고 본체 Collider는 서버 움직임을 그대로 취하게 코드를 변경한다.

```C++
void UGoKartMovementReplicator::SimulatedProxy_OnRep_ServerState()
{
	if (MovementComponent == nullptr)
		return;

	ClientTimeBetweenLastUpdates = ClientTimeSinceUpdate;
	ClientTimeSinceUpdate = 0;

	if (MeshOffsetRoot != nullptr)
	{
		ClientStartTransform.SetLocation(MeshOffsetRoot->GetComponentLocation());
		ClientStartTransform.SetRotation(MeshOffsetRoot->GetComponentQuat());
	}

	ClientStartVelocity = MovementComponent->GetVelocity();

	GetOwner()->SetActorTransform(ServerState.Transform);
}
```

마지막으로 MeshOffset에 맞게 보간 계산을 수정한다.

```C++
void UGoKartMovementReplicator::InterpolateLocation(const FHermiteCubicSpline &Spline, float LerpRatio)
{
	FVector NewLocation = Spline.InterpolateLocation(LerpRatio);
	if (MeshOffsetRoot != nullptr)
	{
		MeshOffsetRoot->SetWorldLocation(NewLocation);
	}
}

void UGoKartMovementReplicator::InterpolateRotation(float LerpRatio)
{
	// 회전 선형보간
	FQuat TargetRotation = ServerState.Transform.GetRotation();
	FQuat StartRotation = ClientStartTransform.GetRotation();

	FQuat NewRotation = FQuat::Slerp(StartRotation, TargetRotation, LerpRatio);

	if (MeshOffsetRoot != nullptr)
	{
		MeshOffsetRoot->SetWorldRotation(NewRotation);
	}
}
```

이를 통해서 MeshOffset과 Collider를 구분하는 근사화 작업을 진행했다.

## 고급 치트 방지

기존에 치트 방지 코드는 Throttle과 SteeringThrow만을 검사하고 있었지만 실제로는 서버에 오게 되는 변조가 가능한 데이터들을 전부 검사해야 올바른 치트 방지를 할 수 있다. 그래서 DeltaTime 등에 서버에 오게되는 시간 정보가 실제 서버 시간보다 빠르다면 이는 변조가 가해진 데이터로 판단해 막는 코드를 구성했다.

코드를 보게 되면 서버에서 관리하는 시간과 클라이언트에서 제시한 DeltaTime을 더해 ProposedTime을 구하고 이를 서버의 현시간<code>GetWorld()->TimeSeconds</code>과 대조해서 만약 이 시간보다 빠르다면 이를 차단한다.

```C++

bool UGoKartMovementReplicator::Server_SendMove_Validate(FGoKartMove Move)
{
	float ProposedTime = ClientSimulatedTime + Move.DeltaTime;
	bool ClientNotRunningAhead = ProposedTime < GetWorld()->TimeSeconds;
	if (!ClientNotRunningAhead)
	{
		UE_LOG(LogTemp, Error, TEXT("Client is running too fast."));
		return false;
	}

	if (!Move.IsValid())
	{
		UE_LOG(LogTemp, Error, TEXT("Received invalid move."));
		return false;
	}

	return true;
}

```

위 코드 중 IsValid의 구현은 아래와 같고 Throttle과 SteeringThrow의 범위를 체크한다.

```C++
	bool IsValid() const
	{
		return FMath::Abs(Throttle) <= 1 && FMath::Abs(SteeringThrow) <= 1;
	}
```

물론 이러한 코드를 구성한다고 모든 치트를 막을 수 있는 것은 아니다. 윈도우 자체 기능을 통한 변조 등 수많은 방법이 있고 안티 치트는 끝없는 막고 뚫는 싸움이기에 최대한 많은 치트를 막을 코드를 끊임없이 고민해야한다.
