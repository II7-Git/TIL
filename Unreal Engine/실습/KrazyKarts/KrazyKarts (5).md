# KrazyKarts (5)

## SimulatedProxy의 움직임 보간

SimulatedProxy에 움직임이 지금은 서버측에 업데이트 타임에 따라서 끊겨서 움직이는 모습을 취하기에 이를 해결하기 위해 업데이트 사이에 움직임을 보간처리하여 이러한 문제를 해결하고자 했다.

먼저 SimulatedProxy는 서버에 정보가 업데이트 될 때 아래의 정보들을 갱신해주었다. 업데이트에 걸린 시간 업데이 된 위치와 속도 정보 등을 가지고 보간 처리를 하여서 SimulatedProxy의 움직임을 부드럽게 구현한다.

현재 SimulatedProxy의 정보를 Start로 잡고 Server에서 업데이트 된 해당 액터의 정보를 Target으로 설정해서 둘 사이에 보간함수들을 적용시키는 방식으로 위치, 속도, 방향 이 세가지 즉, 움직임에 관한 정보들을 보간처리하였다.

```C++
void UGoKartMovementReplicator::SimulatedProxy_OnRep_ServerState()
{
	if (MovementComponent == nullptr)
		return;

	ClientTimeBetweenLastUpdates = ClientTimeSinceUpdate;
	ClientTimeSinceUpdate = 0;

	ClientStartTransform = GetOwner()->GetActorTransform();
	ClientStartVelocity = MovementComponent->GetVelocity();
}
```

### 위치 보간

위치에 대한 보간으로는 <code>FMath::CubicInterp()</code>을 사용했다. CubicInterp은 속도에 따른 도함수도 사용해서 위치를 계산하기에 기존의 선형보간보다 좀 더 자연스러운 보간 구현이 가능해지기에 이를 사용해서 위치 보간을 처리했다.

```C++
void UGoKartMovementReplicator::ClientTick(float DeltaTime)
{
	ClientTimeSinceUpdate += DeltaTime;

	// KINDA_SMALL_NUMBER : 굉장히 작은수를 뜻함 (0.000000001)
	// 너무 작은수로 나누어서 선형보간에서 오류가 생기는 것을 방지
	if (ClientTimeBetweenLastUpdates < KINDA_SMALL_NUMBER)
		return;

	if (MovementComponent == nullptr)
		return;

	// 전후진 속도를 고려한 CubicInterp을 사용한 보간
	FVector TargetLocation = ServerState.Transform.GetLocation();
	float LerpRatio = ClientTimeSinceUpdate / ClientTimeBetweenLastUpdates;
	FVector StartLocation = ClientStartTransform.GetLocation();

	// CubicInterp을 사용하기 위한 도함수 계산
	float VelocityToDerivative = ClientTimeBetweenLastUpdates * 100;
	FVector StartDerivative = ClientStartVelocity * VelocityToDerivative;
	FVector TargetDerivative = ServerState.Velocity * VelocityToDerivative;

	FVector NewLocation = FMath::CubicInterp(StartLocation, StartDerivative, TargetLocation, TargetDerivative, LerpRatio);

	GetOwner()->SetActorLocation(NewLocation);
}
```

### 속도 보간

마찬가지로 ClientTick 에서 구현됐으며 CubicInterp에서 사용된 정보들을 같이 활용해서 새로운 Derivative(도함수)를 구해서 이를 활용해 보간 처리한 속도를 얻어내서 이를 속도로 설정해서 구현해냈다.

```C++
	// 보간된 속도 계산
	FVector NewDerivative = FMath::CubicInterpDerivative(StartLocation, StartDerivative, TargetLocation, TargetDerivative, LerpRatio);
	FVector NewVelocity = NewDerivative / VelocityToDerivative;
	MovementComponent->SetVelocity(NewVelocity);
```

### 방향 보간

방향은 <code>FQuat::Slerp()</code>함수를 이용해 현재 로테이션과 목표하는 서버에서 업데이트 된 로테이션 사이를 보간하여 구현했다.

```C++
	// 회전 선형보간
	FQuat TargetRotation = ServerState.Transform.GetRotation();
	FQuat StartRotation = ClientStartTransform.GetRotation();

	FQuat NewRotation = FQuat::Slerp(StartRotation, TargetRotation, LerpRatio);

	GetOwner()->SetActorRotation(NewRotation);
```

## Replicator의 ClientTick 리팩토링

ClientTick에서 다양한 보간계산을 전체적으로 하다보니 하나의 함수가 규모가 너무 커져서 이를 리팩토링해서 코드를 깔끔하게 정리했다.

먼저 보간에서 쓰일 정보들을 구조체로 만들어서 관리하기로 했고 이를 헤더 파일에 정의해놓았다.

<code>StartLocation, StartDerivative, TargetLocation, TargetDerivative</code> 보간에서 쓰였던 4개의 변수와 이를 통해 구현하는 FMath 함수를 구조체의 넣어서 구성했다.

```C++
struct FHermiteCubicSpline
{
	FVector StartLocation, StartDerivative, TargetLocation, TargetDerivative;

	FVector InterpolateLocation(float LerpRatio) const
	{
		return FMath::CubicInterp(StartLocation, StartDerivative, TargetLocation, TargetDerivative, LerpRatio);
	}
	FVector InterpolateDerivative(float LerpRatio) const
	{
		return FMath::CubicInterpDerivative(StartLocation, StartDerivative, TargetLocation, TargetDerivative, LerpRatio);
	}
};

```

그 뒤 기존 ClientTick에서는 위에서 만든 구조체를 사용해서 코드 리팩토링을 진행했다.

```C++
void UGoKartMovementReplicator::ClientTick(float DeltaTime)
{
	ClientTimeSinceUpdate += DeltaTime;

	// KINDA_SMALL_NUMBER : 굉장히 작은수를 뜻함 (0.000000001)
	// 너무 작은수로 나누어서 선형보간에서 오류가 생기는 것을 방지
	if (ClientTimeBetweenLastUpdates < KINDA_SMALL_NUMBER)
		return;

	if (MovementComponent == nullptr)
		return;
	float LerpRatio = ClientTimeSinceUpdate / ClientTimeBetweenLastUpdates;
	float VelocityToDerivative = ClientTimeBetweenLastUpdates * 100;

	FHermiteCubicSpline Spline = CreateSpline();

	InterpolateLocation(Spline, LerpRatio);

	InterpolateVelocity(Spline, LerpRatio);

	InterpolateRotation(LerpRatio);
}
```

CreateSpline

```C++
FHermiteCubicSpline UGoKartMovementReplicator::CreateSpline()
{
	FHermiteCubicSpline Spline;
	// 전후진 속도를 고려한 CubicInterp을 사용한 보간
	Spline.TargetLocation = ServerState.Transform.GetLocation();
	Spline.StartLocation = ClientStartTransform.GetLocation();

	// CubicInterp을 사용하기 위한 도함수 계산

	Spline.StartDerivative = ClientStartVelocity * VelocityToDerivative();
	Spline.TargetDerivative = ServerState.Velocity * VelocityToDerivative();

	return Spline;
}

```

InterpolateLocation

```C++
void UGoKartMovementReplicator::InterpolateLocation(const FHermiteCubicSpline &Spline, float LerpRatio)
{
	FVector NewLocation = Spline.InterpolateLocation(LerpRatio);
	GetOwner()->SetActorLocation(NewLocation);
}
```

InterpolateVelocity

```C++
void UGoKartMovementReplicator::InterpolateVelocity(const FHermiteCubicSpline &Spline, float LerpRatio)
{

	FVector NewDerivative = Spline.InterpolateDerivative(LerpRatio);
	FVector NewVelocity = NewDerivative / VelocityToDerivative();
	MovementComponent->SetVelocity(NewVelocity);
}

```

InterpolateRotation

```C++
void UGoKartMovementReplicator::InterpolateRotation(float LerpRatio)
{
	// 회전 선형보간
	FQuat TargetRotation = ServerState.Transform.GetRotation();
	FQuat StartRotation = ClientStartTransform.GetRotation();

	FQuat NewRotation = FQuat::Slerp(StartRotation, TargetRotation, LerpRatio);

	GetOwner()->SetActorRotation(NewRotation);
}
```
