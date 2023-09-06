# CryptRaider 1

## 기본 세팅 및 Asset

프로젝트는 아래 사진과 같이 1인칭 프로젝트를 통해 제작한다.<br>
이는 캐릭터 조작을 1인칭 기반으로 할 것이기 때문이다.

![1](/Assets/Images/Unreal/실습/CryptRaider/1.png)

사용할 에셋은 마켓 플레이스에 존재하는 무료 에셋인 Medival Dungeon을 사용한다

![2](/Assets/Images/Unreal/실습/CryptRaider/2.png)

## 레벨 구성

레벨은 아래와 같이 구성을 한다

![3](/Assets/Images/Unreal/실습/CryptRaider/3.png)

인 게임으로 보면 이러한 느낌의 던전을 구성하게 된다

![4](/Assets/Images/Unreal/실습/CryptRaider/4.png)

해당 내용을 정리할 때 사용된 중요한 내용들은 아래 링크에 정리해두었다.

[레벨 구성 시 TIP](./../../이론%20및%20정리/레벨%20구성%20시%20TIP.md)

## 라이트 구성

라이트 기능에 대한 자세한 정리는 아래 글에 해놓았다.

[라이트(Light) 기능 정리](<../../이론%20및%20정리/라이트(Light)%20정리.md>)

이를 바탕으로 환경 조명을 구성한 후 Torch를 수정하여 맵에 잘 배치해주어서 빛이 새지 않는 외벽 구조를 만든 후에 안에 조명 구성을 완료했다.

![4](/Assets/Images/Unreal/실습/CryptRaider/5.png)
![4](/Assets/Images/Unreal/실습/CryptRaider/6.png)
![4](/Assets/Images/Unreal/실습/CryptRaider/7.png)

## Mover 벽 생성

암호를 해결하면 문이 열리는 어드벤쳐 게임 형식이기에 암호가 풀렸을 때 열리는 조건을 가진 문이 필요하다. 그래서 Mover라는 클래스를 만들어 벽을 이동시키는 것을 구현했는데 이때 유용하게 쓰일 함수가 있다. 그 함수에 공식 문서를 참조한다.

[FMath::VInterpConstantTo](https://docs.unrealengine.com/5.2/en-US/API/Runtime/Core/Math/FMath/VInterpConstantTo/)

해당 함수의 원형을 보면 current위치를 target위치까지 일정한 스피드로 이동시키는 계산을 하여 FVector를 반환하는 함수인데 이를 잘 사용하면 쉽게 벽 이동을 구현할 수 있다.

구현 예시

```C++
if (ShouldMove)
	{
		FVector CurrentLocation = GetOwner()->GetActorLocation();
		FVector TargetLocation = OriginalLocation + MoveOffset;
		float Speed = FVector::Distance(OriginalLocation, TargetLocation) / MoveTime;

		FVector NewLocation = FMath::VInterpConstantTo(CurrentLocation, TargetLocation, DeltaTime, Speed);
		GetOwner()->SetActorLocation(NewLocation);
	}
```

위 처럼 Speed를 적절히 구해주고 목표 위치와 현재 위치를 계산 후 새로운 로케이션을 계산해 그 위치로 현재 액터를 변경해주면 원하는 기능을 구현할 수 있다.

![8](/Assets/Images/Unreal/실습/CryptRaider/8.png)
![9](/Assets/Images/Unreal/실습/CryptRaider/9.png)
