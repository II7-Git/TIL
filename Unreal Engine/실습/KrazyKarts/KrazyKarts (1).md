# KrazyKarts (1)

## 카트 이동 구현

### 카트의 전진 구현

기존에 단순히 입력값에 따른 이동 구현은 자동차에 움직임과는 괴리감이 있기에 힘을 계속해서 가해주어서 가속도를 얻어서 이를 통해서 이동 속도에 따른 이동처리를 하게끔 변경했다.<br>
그렇기에 입력값에 따른 가속도를 구해서 이를 바탕으로 질량\*가속도를 해서 이동을 구현했다.

아래에 코드에 속도를 구하는 물리적인 방식을 볼 수 있다.

```C++
// 힘은 방향 벡터* 최대 속력* 스로틀(1.0~-1.0) 값으로 가해질 수 있는 최대 힘의 절댓값 내에서 동작할 수 있게 설정됐다.
FVector Force = GetActorForwardVector() * MaxDrivingForce * Throttle;

// F=M*A 의 법칙에 의해 가속도는 힘/질량으로 구해진다.
FVector Acceleration = Force / Mass;


// 최종 속도는 현재 속도+ (가속도* 시간)으로 구한다.
Velocity = Velocity + Acceleration * DeltaTime;
```

이렇게 얻은 속도 값을 통해서 이동을 구현한다.

코드는 아래와 같고 이동 벡터는 속도와 시간을 곱해서 구하고 이를
AddActorWorldOffset(Translation, true, &Hit) 의 인자로 써서 이동 시킨다.

이때 true 옵션은 해당 위치로 이동하기 전에 미리 위치로 이동가능한지 체크하는 Sweep 옵션을 킨다는 뜻이고 이를 통해 콜리전 충돌여부를 따져서 이동 불가한 위치면 가지 않게 설정한다. Hit은 FHitResult 의 포인터로 이 곳에 Sweep에서 따진 충돌 결과를 외부 변수 Hit에 저장하게 된다. 포인터이기에 만약 필요없다면 null로 넘길 수 있다<br>

만약 Hit.IsValidBlockingHit()을 통해 차가 무언가 충돌해 차가 정지했다면 이동 속도를 0으로 만들어주어서 다음 이동에 영향을 주지 않게끔 설정했다.

```C++
void AGoKart::UpdateLocationFromVelocity(float DeltaTime)
{
	FVector Translation = Velocity * 100 * DeltaTime;

	FHitResult Hit;
	AddActorWorldOffset(Translation, true, &Hit);

	if (Hit.IsValidBlockingHit())
	{
		Velocity = FVector::ZeroVector;
	}
}
```

### 카트의 회전 구현

회전을 시키는 방법으로 기존에는 FRotator를 많이 썼지만 이번에는 FQuat을 사용했다.

FQuat의 장점은 특정 축(벡터)와 그 축을 기준으로 회전하려는 각도만으로 회전을 시킬 수 있다는 점과 그렇기에 X,Y,Z를 통한 고정 위치에 따른 회전이 아닌 자유로운 회전이 가능하다는 점에 있다.

참고 : [FQuat DOCS](https://docs.unrealengine.com/4.26/en-US/API/Runtime/Core/Math/FQuat/)

이를 통해서 회전을 구하여 AddActorWorldRotation()을 통해서 회전을 적용시켜준다.<br>
이때 회전하는 만큼 속도 벡터도 방향을 회전시켜야 차의 전진방향과 회전방향을 일치시킬 수 있게된다.

그에 따라 구현한 코드는 아래와 같다.

```C++
void AGoKart::ApplyRotation(float DeltaTime)
{
	// 회전 적용
	float RotationAngle = MaxDegreesPerSecond * DeltaTime * SteeringThrow;
	// FRotator로는 못하는 여러 축에 따른 회전이 가능
	// 현재 액터의 업 벡터에서 라디안 만큼 회전
	// 따라서 RotationAngle을 라디안으로 변환
	FQuat RotationDelta(GetActorUpVector(), FMath::DegreesToRadians(RotationAngle));

	// 현재 전진하는 벡터를 회전하는 FQuat만큼 회전시켜서 회전각도와 자동차의 전진 각도가 일치하게 하여 이동에 어색함을 없게끔한다.
	Velocity = RotationDelta.RotateVector(Velocity);

	AddActorWorldRotation(RotationDelta);
}
```

### 카트의 최대 속도와 공기 저항

자동차의 최대 속도가 존재하는 이유는 자동차의 속력에 따라 받게 되는 공기 저항이 커지기 때문이다.

공기 저항식은 아래와 같다. 아래에서 확인할 수 있든 속력의 제곱에 비례하여 공기저항이 커지기 때문에 이 값이 속도와 반대로 적용되서 맞부딪혀 가속도가 0이 되는 순간 더 이상 속력을 올릴 수 없는 최대 속도가 되게된다.<br>DragCoefficient는 저항계수를 뜻하는데 차체의 설계 등에 따라서 바뀔 수 있는 값으로 차체의 설계는 이 값을 줄이기 위해 설계한다고 생각하면 편하다.

```
AirResistance = - Speed² * DragCoefficient
```

위 식을 적용해서 실질적으로 저항을 얻으면 아래와 같다.

```C++
Fvector AGoKart::GetResistance()
{
	return -Velocity.GetSafeNormal() * Velocity.SizeSquared() * DragCoefficient;
}

```

위의 값을 공기 저항으로 적용시켜준다.

```C++
// 공기 저항 적용
Force += GetResistance();
```

### 구름 저항(Rolling Resistance)

구름 저항은 바퀴같은 구르는 물체에 적용되는 저항으로 이를 통해서 공기 저항만으로는 쉽게 멈추지 않는 자동차에 저항을 줘서 좀 더 자연스러운 감속을 구현하려고 한다.

구름 저항식은 아래와 같은데 NormalForce는 중력에 대해서 작용 반작용으로 땅이 밀어올리는 힘을 뜻한다. 따라서 중력이 강해질수록 NormalForce도 강해지게 된다.

```
RollingResistance= RRCoefficient x NormalForce
```

이를 통해 구름 저항을 적용시키는 코드는 아래와 같다

```C++
FVector AGoKart::GetRollingResistance()
{
	// 월드에 설정된 중력값 알아내기
	float AccelerationDueToGravity = -GetWorld()->GetGravityZ() / 100;
	// 중력에 대항하는 NormalForce 계산 // NormalForce= M(질량)*G(중력)
	float NormalForce = Mass * AccelerationDueToGravity;
	return -Velocity.GetSafeNormal() * RollingResistanceCoefficient * NormalForce;
}
```

### Steering 설정

현재 상태는 차가 멈춰있어도 회전이 가능한 실제 차 움직임과는 괴리가 있는 상태이다. 이를 차의 회전 반경에 맞춰서 회전하는 Steering을 구현해서 움직임을 바꾸려고 한다.

차의 회전 반경을 일정한 축을 기준으로 정해진 반지름만큼의 거리를 두고 회전하는 원의 형태를 가지기에 이를 코드에 적용시켜 속도에 맞춰서 차가 회전 반경에 따라서 회전하는 모습을 구현할 수 있다. 그 코드를 구현하면 아래와 같다.

```C++
// 회전 반경을 고려한 회전 구현
void AGoKart::ApplyRotation(float DeltaTime)
{

	// 속력에 시간을 곱해서 이동 거리 계산
	// 속도 벡터와 액터의 전방 벡터를 내적하여 후진할 때 상황에도 대응할 수 있는 속력값 얻어내기
	// EX ) 전진이라면 +, 후진이라면 -값을 리턴하게된다.
	float DeltaLocation = FVector::DotProduct(GetActorForwardVector(), Velocity) * DeltaTime;
	// 실제 회전 각도 = 이동 거리/회전 반경*핸들의 회전각도
	float RotationAngle = DeltaLocation / MinTurningRadius * SteeringThrow;
	// FRotator로는 못하는 여러 축에 따른 회전이 가능
	// 현재 액터의 업 벡터에서 라디안 만큼 회전
	FQuat RotationDelta(GetActorUpVector(), RotationAngle);

	// 현재 전진하는 벡터를 회전하는 FQuat만큼 회전시켜서 회전각도와 자동차의 전진 각도가 일치하게 하여 이동에 어색함을 없게끔한다.
	Velocity = RotationDelta.RotateVector(Velocity);

	AddActorWorldRotation(RotationDelta);
}
```

위 코드를 보게 되면 DeltaLocation은 차가 이동하는 거리인데 뒤로 가는 후진 상황에도 대응하기 위해 단순 속력이 아닌 전방 벡터와 속도의 내적으로 스칼라 값을 얻고 이에 시간을 곱해주는 것이 중요한 점이다. 이렇게 하여 후진 상황에는 -값, 전진에는 +값을 얻어서 올바르게 처리가 가능해진다.<br>

다음 회전 각도(Rotation Angle)는 아래의 식을 이용하면 구할 수 있다.

```
원에서 이동 거리= 각도*반지름
```

이를 활용해서 DeltaLocation/MinTurningRadius(회전 반경의 반지름)\*SteeringThrow(핸들 방향) 을 해주게 된다면 실제로 차가 바퀴 축에 따라서 속도에 맞춰서 회전해야하는 각도를 얻을 수 있게 된다. 그래서 결국 자연스러운 회전 움직임을 얻어낼 수 있다.
