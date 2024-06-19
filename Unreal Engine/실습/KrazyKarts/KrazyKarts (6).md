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
