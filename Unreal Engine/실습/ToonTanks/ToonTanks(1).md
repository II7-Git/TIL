# ToonTanks Project

툰탱크 프로젝트를 진행하면서 배우게 된 지식들을 간단히 기록해보겠다.

## 전방 선언

해당 내용에 대해서는 아래 링크에 적어놓았다.

[전방 선언 정리](/Unreal%20Engine/이론%20및%20정리/언리얼에서%20전방%20선언과%20사용%20이유.md)

## UPROPERTY 옵션

### 카테고리 지정

옵션에 Category="원하는 이름" 을 넣게 된다면
원하는 이름 카테고리 밑으로 변수가 들어가서 표시되게 된다. 이는 카테고리를 통한 변수 찾기 및 정리에 용이하기 때문에 유용하다

### 변수 노출

지금껏 습관처럼 사용하던 UPROPERTY에서 옵션인 EditAnywhere 같은 것에 실질적인 적용 예와 이벤트 그래프에서 사용할 수 있는 옵션에 대해서 알아보았다.

아래 표에 따라서 실제 사용 목적에 따라서 권한을 조절하는 식으로 작동시키면 된다.
![2](/Assets/Images/Unreal/실습/ToonTanks/2.png)

```C++
// 블루프린트와 인스턴스 어디서든 사용가능하며, 이벤트 그래프에서도 get,set이 가능하다
UPROPERTY(EditAnywhere, BlueprintReadWrite)
	float Speed = 400.0f;
```

### 만약 private에 속해있는 변수를 노출시키고 싶다면?

그럴 경우에는 UPROPERTY()에 meta=(AllowPrivateAccess = "true") 옵션을 넣어주면 된다.

### 최종 정리

![3](/Assets/Images/Unreal/실습/ToonTanks/3.png)

실제 예

```C++
private:
//블루프린트 ,인스턴스 둘 다 위치에서 보이기만 하며 이벤트 그래프에서 get이 가능하고 카테고리는 "Components" 밑에 소속되어있으며 private에 있어도 표시가 가능하다
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components", meta = (AllowPrivateAccess = "true"))
	class UCapsuleComponent *CapsuleComp;
```

해당 기능을 사용하여 실제 블루프린트로 탱크와 터렛을 만들어 배치시킨 모습
![3](/Assets/Images/Unreal/실습/ToonTanks/4.png)
