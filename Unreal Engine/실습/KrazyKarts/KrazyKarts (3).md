# KrazyKarts (3)

## 클라이언트 예측 구현

서버에서 업데이트 되는 정보는 분명한 렉 등의 이유 등으로 한계가 있기에 이 업데이트되는 정보를 클라이언트에서 문제없이 동작시키는 방식이 필요하다. 그래서 필요한 것이 큐를 통한 동작의 대기열을 구현해서 서버측에서 받은 데이터를 자연스럽게 클라이언트가 시뮬레이션 시키는 방법을 구현해야한다.

먼저 기존에는 Velocity,Steering 등 여러 정보를 혼재해서 업데이트 했는데 이 정보를 언리얼 구조체로 만들어서 모두 하나의 객체로 관리하게 변형시켰다.

<code>FGoKartMove</code>는 움직임에 관련된 정보들을 담고 있고 <code>FGoKartState</code> 이 Move와 더불어 위치,속도 정보까지 가지고 있다.

추가로 <code>FGoKartMove</code> 에 있는 Time 변수를 통해서 클라이언트에서 예측과 서버의 업데이트 정보가 틀어져서 다시 과거로 되돌아가는 현상을 방지하게 된다.

```C++
/* 이동에 관련된 변수들을 구조체로 만들어서 클라이언트 자체적으로 이동을 처리하다가
서버에서 이 구조체에 대한 정보가 오면 기존 구조체들을 버리고 서버에서 온 정보로 업데이트하여
자연스러운 동기화 구현 */
/*
서버에서 온 정보를 정확히 파악하는 방법은 서버에서 보내는 정보에 일련번호(sequence Number)를 사용하는 방법등이 있다.
이번에는 Time 변수를 통해서 서버에서 실행된 움직임 처리의 시간을 통해서 클라이언트에서 동기화 작업을 진행한다.
즉 Time보다 오래된 클라이언트측의 Move는 지우면서 동기화한다.
*/
USTRUCT()
struct FGoKartMove
{
	GENERATED_BODY()

	UPROPERTY()
	float Throttle;
	UPROPERTY()
	float SteeringThrow;

	UPROPERTY()
	float DeltaTime;

	UPROPERTY()
	float Time;
};

USTRUCT()
struct FGoKartState
{
	GENERATED_BODY()

	UPROPERTY()
	FTransform Transform;

	UPROPERTY()
	FVector Velocity;

	UPROPERTY()
	FGoKartMove LastMove;
};
```

## 승인되지 않은 대기열을 통한 자체 움직임 및 서버 업데이트

아까 말했듯이 대기열을 통해서 클라이언트는 자체적인 시뮬레이션을 돌려서 서버에 업데이트되는 정보 사이에 자신만의 자연스러운 움직임을 취해야한다.<br/>
또 그러면서도 서버에서 정보가 오게됐을 때 그 정보에 따라 동작을 업데이트 할 때 자연스럽게 동작해야한다.

이를 위해서 대기열을 사용하게 된다.

아래와 같이 승인되지 않은 대기열을 만들어서 클라이언트는 서버에 상관없이 자신의 동작을 처리하게 할 것이다.

```c++
TArray<FGoKartMove> UnacknowledgedMoves;
```

이제 CPP에서의 흐름을 볼건데 먼저 Tick 함수에서의 동작은 아래와 같다.

자율 프록시는 조작에 따른 Move를 생성 후 이를 자체적으로 시뮬레이트하고 대기열에 해당 Move를 넣어두고 서버 측에도 해당 Move에 대한 정보를 보낸다.

만약 서버에서도 동작시키는 폰이 있다면 이 때는 차피 서버의 정보가 그대로 폰과 일치하기에 굳이 대기열이나 자체 시뮬레이트 없이 <code>Server_SendMove(Move);</code>를 통해서 실행시키면 된다.

추가로 자율 프록시가 아닌 즉 조작하지 않는 폰들은 매 틱마다 서버의 마지막 정보를 가지고 자율 예측을 진행한다.

```C++

void AGoKart::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

	if (GetLocalRole() == ROLE_AutonomousProxy)
	{
		FGoKartMove Move = CreateMove(DeltaTime);
		SimulateMove(Move);

		UnacknowledgedMoves.Add(Move);
		Server_SendMove(Move);
	}

	// 서버이면서 동시에 특정 Pawn을 조종하고 있는 경우
	if (GetLocalRole() == ROLE_Authority && IsLocallyControlled())
	{
		FGoKartMove Move = CreateMove(DeltaTime);
		Server_SendMove(Move);
	}

	/*
	각 클라이언트에서 조종하고 있지 않은 폰들이 OnRep_ServerState로 업데이트된
	정보에 따라서 예측해서 동작을 취하여 자연스럽게 움직이게 하기
	*/
	if (GetLocalRole() == ROLE_SimulatedProxy)
	{
		SimulateMove(ServerState.LastMove);
	}
	DrawDebugString(GetWorld(), FVector(0, 0, 100), GetEnumText(GetLocalRole()), this, FColor::White, DeltaTime);
}

```

그럼 Tick에서 사용된 Server_SendMove를 보면 클라이언트가 보낸 움직임(Move)를 가지고 서버에서도 폰의 움직임을 처리하고 서버 측에서 관리하는 ServerState에 서버에서 처리한 정보들을 저장하게 된다.

```C++

void AGoKart::Server_SendMove_Implementation(FGoKartMove Move)
{
	SimulateMove(Move);

	ServerState.LastMove = Move;
	ServerState.Transform = GetActorTransform();
	ServerState.Velocity = Velocity;
	// TODO  :  Update Last Move
}
```

이렇게 <code>ServerState</code>가 수정됐으면 레플리케이션이 동작해서 클라이언트들에 ServerState가 업데이트 돼고 그에 따라 <code>OnRep_ServerState()</code> 함수도 실행되는데 해당 기능은 아래와 같다.

<code>ServerState</code>의 정보를 가지고 클라이언트의 위치 정보를 업데이트하고 <code>ClearAcknowledgedMoves(ServerState.LastMove)</code> 를 통해서 서버에서 업데이트 된 마지막 Move를 기점으로 현재 클라이언트가 가지고 있는 대기열을 재구성하게 된다. 이 때 기준은 Move의 있는 Time을 기준으로 서버에서 업데이트 된 마지막 Move보다 오래된 동작들은 삭제하여 폰이 서버에서 업데이트 된 위치에서 과거의 위치로 점프하는 현상을 방지하게 한다. <br/>

그 뒤 대기열의 남은 Move를 전부 시뮬레이트하여서 아직 남은 동작을 자체 시뮬레이트하게 된다.<br/>
이렇게 하면 약간의 지연을 될지언정 뒤로 돌아가는 등의 부자연스러운 움직임을 자체 클라이언트에서 취하지는 않을 수 있게 된다.

```C++
void AGoKart::OnRep_ServerState()
{
	SetActorTransform(ServerState.Transform);
	Velocity = ServerState.Velocity;

	ClearAcknowledgedMoves(ServerState.LastMove);

	for (const FGoKartMove &Move : UnacknowledgedMoves)
	{
		SimulateMove(Move);
	}
}
```

추가로 <code>ClearAcknowledgedMoves</code>의 구현을 확인하면 아래와 같다. 앞서 말했듯이 Time을 기준으로 서버에 마지막 Move보다 오래된 동작들은 대기열에서 삭제하여 불필요한 과거 동작이 작동하지 않게 한다.

```C++
/*
서버에서 새로운 Move 정보가 왔을 때 이 Move보다
동작시간이 오래된 Move들을 지워서 되돌아가는 현상 방지
*/
void AGoKart::ClearAcknowledgedMoves(FGoKartMove LastMove)
{
	TArray<FGoKartMove> NewMoves;

	for (const FGoKartMove &Move : UnacknowledgedMoves)
	{
		if (Move.Time > LastMove.Time)
		{
			NewMoves.Add(Move);
		}
	}

	UnacknowledgedMoves = NewMoves;
}
```

이제 이렇게 클라이언트에서 자체적인 동작을 자연스럽게 시뮬레이트하는 것은 구현했고 남은 건 자율 프록시가 아닌 시뮬레이트 프록시들의 예측을 보간해서 다른 폰들도 자연스럽게 움직이는 것을 클라이언트에서 구현하는 일이다.
