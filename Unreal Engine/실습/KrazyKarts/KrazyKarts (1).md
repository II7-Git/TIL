# KrazyKarts (1)

## 카트 이동 구현

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

이때 true 옵션은 해당 위치로 이동하기 전에 미리 위치로 이동가능한지 체크하는 Sweep 옵션을 킨다는 뜻이고 이를 통해 콜리전 충돌여부를 따져서 이동 불가한 위치면 가지 않게 설정한다. Hit은 FHitResult로 이 곳에 Sweep에서 따진 충돌 결과를 외부 변수 Hit에 저장하게 된다.<br>

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
