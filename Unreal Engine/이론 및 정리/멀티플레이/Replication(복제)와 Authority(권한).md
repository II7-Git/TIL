## Replication

서버가 어떠한 상태를 처리 후 그 결과를 모든 클라이언트에게 전하는 것을 Replication(복제)라고 한다. 이것이 제대로 되어야 클라이언트에서 게임을 제대로 진행할 수 있다.

replicate 된 액터와 프로퍼티를 서버가 설정해야 클라이언트는 해당 서버에서의 액터의 상태를 제대로 받아서 복제된 액터에 적용시킬 수 있다.

- 액터와 Movement 상태를 Replicate에 등록하는 코드 예시(BeginPlay에서 적용시켜야한다.)
- Authority를 가지고 있는 서버에서만 동작시키도록 하는 것이 올바르다.
- 아래처럼 적용시키면 클라이언트에 해당 액터와 움직임이 복제되어 전달되서 클라이언트에서도 동일한 동작을 하게된다.

```C++
void AMovingPlatform::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        SetReplicates(true);
        SetReplicatedMovement(true);
    }
}
```

## Authority

Replication의 핵심 개념으로써 서버와 클라이언트를 구분하게 해주는 개념으로 권한을 가지고 있다면 서버 아니라면 클라이언트이다.
