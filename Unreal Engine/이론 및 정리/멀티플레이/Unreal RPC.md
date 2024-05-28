# [RPC](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/remote-procedure-calls-in-unreal-engine)

## RPC란?

언리얼에서 서버-클라이언트 구조로 구동을 할 때 클라이언트 측에서 아무리 클라이언트 요소를 조작해봤자 서버는 이를 인지하고 처리할 수 없다.

그렇기에 이를 처리하기 위해서 클라이언트는 내가 어떠한 조작을 했고 이를 처리해주기를 서버측에 요청을 해야하는데 이것이 RPC 함수이다.<br>즉 RPC함수는 서버-클라이언트 구조에서 통신이 가능하게 하는 함수를 뜻한다.

## RPC 사용하기

RPC를 언리얼에서 사용할 땐 UFUNCTION 선언 뒤에 Server,Client,NetMulticast 키워드를 붙여서 사용한다.

서버측에서 실행시켜 클라이언트에서 실행될 때는 Client를 쓴다.

```C++
	UFUNCTION( Client )
	void ClientRPCFunction();
```

클라이언트측에서 서버에 실행시키길 요청하는 함수는 Server를 쓴다.

```C++
	UFUNCTION( Server )
	void ServerRPCFunction();
```

Multicast는 특수 유형인데 서버에서 호출된 뒤 서버는 모든 클라이언트에서도 해당 함수가 실행되게 한다.

```C++
	UFUNCTION( NetMulticast )
	void MulticastRPCFunction();
```

## 실제 예

헤더 파일에서 서버측에 반영되고 싶은 함수를 아래와 같이 Server키워드로 선언한다.

```C++
// ServerRPC임을 명시
UFUNCTION(Server, Reliable, WithValidation)
void Server_MoveForward(float Value);
```

Server 뒤에 두 키워드는 각각의 역할이 있고 Server RPC를 사용하기 위해서 꼭 사용되야 하는데 그 사용법과 이유는 아래 적혀있다.

```
Reliable 키워드?

기본적으로 RPC는 비신뢰성을 가지고 있기에 서버측에서 확실히 실행시키기 위해서는 Reliable 키워드를 통해 신뢰성을 확보해야한다.
```

```
WithValidation 키워드?

서버는 클라이언트 측에 요청에 악성 데이터, 잘못된 입력, 치트 등에 대해서 항상 방지 대책이 존재해야한다. 그래서 서버 측에서는 Server RPC에 대해서 인증 과정을 거치고 만약 인증을 통과하지 못하면 연결을 끊게 하는 구조를 가지는데 그러기 위해 선언하는 키워드가 WithValidation이다.
```

CPP 파일에서 실제 구현은 기존의 구현과는 조금 다르게 된다.

먼저 실제 서버에서 실행될 내용을 Implementation 부분에서 구현한다.<br> 아래처럼 기존 함수 이름 뒤에 \_Implementation을 붙이고 이 부분에서 구현을 하면 된다.

```C++
void AGoKart::Server_MoveForward_Implementation(float Value)
{
	Throttle = Value;
	// Velocity = GetActorForwardVector() * 20 * Value;
}
```

WithValidation 키워드를 넣었기에 Server_MoveForward의 검증 함수도 필요하다 이때 키워드는 \_Validate이다.<br>
아래 같은 경우 간단하게 입력값이 1을 넘지 않으면 통과이고 만약 넘는 치트행위가 들어오면 서버-클라이언트 연결은 끊기게 된다.

```C++
// MoveForward의 유효성 검증으로 치트 방지
bool AGoKart::Server_MoveForward_Validate(float Value)
{
	return FMath::Abs(Value) <= 1;
}
```

마지막으로 실제 다른 코드에서 사용은 기존의 함수 이름인 Server_MoveForward만 가지고 사용하면 된다. 뒤에 붙은 \_Implementation 같은 부분은 사용해선 안된다.

```C++
void AGoKart::SetupPlayerInputComponent(UInputComponent *PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

	PlayerInputComponent->BindAxis("MoveForward", this, &AGoKart::Server_MoveForward);
}

```

이러한 방식으로 서버-클라이언트 구조에서 통신 가능한 함수를 사용할 수 있게 된다.
