# KrazyKarts (2)

## 서버-클라이언트 Unreal RPC

서버-클라이언트 구조에서 서버 측에서 클라이언트에 조작에 따른 요청을 받고 처리해서 다시 반영하기 위해서는 통신을 위한 특별한 함수를 사용해야 하는데 이것이 RPC이다 이에 대해서 자세한 기능 정리는 아래 링크에 해놓았다.

#### [Unreal RPC](/Unreal%20Engine/이론%20및%20정리/멀티플레이/Unreal%20RPC.md)

이에 따른 구현 부분은 아래와 같고 \_Implementation 키워드와 \_Validate를 각각 구현한 것을 확인할 수 있다.

```C++
void AGoKart::Server_MoveForward_Implementation(float Value)
{
	Throttle = Value;
	// Velocity = GetActorForwardVector() * 20 * Value;
}

// MoveForward의 유효성 검증으로 치트 방지
bool AGoKart::Server_MoveForward_Validate(float Value)
{
	return FMath::Abs(Value) <= 1;
}
```

## SimulatedProxy & AutonomousProxy

레플리케이션 방식을 이용하여 액터들을 조작할 때 중요한 프로퍼티가 있는데 그것이 바로 롤(Role)이다.

그에 대해서 자세한 정리는 아래 링크에 해놓았다.

#### [액터 롤 및 리모트 롤](/Unreal%20Engine/이론%20및%20정리/멀티플레이/액터%20롤%20및%20리모트%20롤.md)

위에 내용에 따라서 우리는 서버 측에서 모든 액터에 Authority 권한이 있는 것과 클라이언트 측에서 플레이어 컨트롤러로 조작하는 객체는 AutonomousProxy ,그 외 객체들은 SimulatedProxy가 되는 것을 확인할 수 있다.

AActor에 있는 GetLocalRole(), GetRemoteRole() 사용해서 롤과 리모트 롤을 모두 알아낼 수 있다.

이를 통해서 실제 가동시키면 아래와 같이 확인해볼 수 있다.<br>
왼쪽 서버에서는 모든 자동차 객체가 Authority인 반면, 왼쪽 클라이언트는 자신의 자동차는 AutonomousProxy이고 상대 자동차는 SimulatedProxy인 것을 확인할 수 있다.
![1](/Assets/Images/Unreal/실습/KrazyKarts/1.png)

이를 통해서 우리는 AutonomousProxy인 객체를 보다 자연스럽게 조작이 가능해진다.

기존의 코드에서 변형을 하면 아래와 같다. 플레이어 컨트롤러에 영향을 받는 AutonomousProxy 객체의 조작은 MoveForward나 MoveRight로 하고 이 값과 상관없이 Server측에 다시 컨트롤러로 발동 된 함수에서 RPC를 보내는 방식으로 내가 컨트롤하는 객체는 보다 자연스러운 움직임이 가능해진다.

```C++
void AGoKart::SetupPlayerInputComponent(UInputComponent *PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

	PlayerInputComponent->BindAxis("MoveForward", this, &AGoKart::MoveForward);
	PlayerInputComponent->BindAxis("MoveRight", this, &AGoKart::MoveRight);
}

void AGoKart::MoveForward(float Value)
{
	Throttle = Value;
	Server_MoveForward(Value);
}
void AGoKart::MoveRight(float Value)
{
	SteeringThrow = Value;
	Server_MoveRight(Value);
}
```

만약 내 컨트롤러에서 부정한 치트를 써서 값을 조작하려 해도 Server Validate 함수가 제대로 구현되어있다면 서버 측에서 연결을 거부할 것이기 때문에 이 부분에 대해서는 문제는 없다.

단, 진짜 문제는 이렇게 조작을 좀 해보면 객체가 서버와 클라이언트 위치에서 어긋나있는 것을 확인할 수 있는데 이 이유는 서버와 클라이언트가 각각 작동되는 프레임 수가 다르기에 DeltaTime이 어긋나서 생기는 에러, 혹은 그 밖에 다른 에러들이 겹겹이 쌓여서 생기는 에러 때문에 서로 다른 상태로 표현되는 것이 가장 큰 문제이다.

그렇기에 이러한 문제를 해결하기 위한 동기화 및 보간 작업이 필수적이다.
