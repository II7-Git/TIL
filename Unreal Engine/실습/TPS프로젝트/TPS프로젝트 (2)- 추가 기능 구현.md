# TPS프로젝트 (2)- 추가 기능 구현

## 추가 기능 : 달리기

달리기 기능을 어떻게 할까 고민하다 기존 게임에서 많이 사용하는 Shift키를 활용한 토글 방식의 달리기를 구현하고자 하였습니다.

그래서 해당 키를 먼저 등록해보겠습니다. 아래처럼 에셋을 추가했습니다.

![1](/Assets/Images/Unreal/실습/TPSProject/2.추가기능/1.png)

다음 IMC_Default에 IA_Run을 등록하고 '왼쪽 Shift'를 등록합니다.

![2](/Assets/Images/Unreal/실습/TPSProject/2.추가기능/2.png)

다음은 코드에 추가될 내용입니다. 내용은 추가된 부분들만 적겠습니다.

### TPSProjectCharacter.h

```C++

	//달리기 버튼 추가
	/** Run Input Action */
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Input, meta = (AllowPrivateAccess = "true"))
	class UInputAction* RunAction;

protected:

    // 달리기 토글값 등록할 변수
	bool bRunning;
	void Run();
    // 이동이 멈추면? 이 때도 달리기가 멈춰야한다
	void StopRun();
```

### TPSProjectCharacter.cpp

추가된 부분들만 작성했으니 참고부탁드립니다.

```C++
ATPSProjectCharacter::ATPSProjectCharacter()
{
	//달리기 초기화
	bRunning = false;
}

void ATPSProjectCharacter::SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent)
{
	// 동작들 등록
	// Set up action bindings
	if (UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(PlayerInputComponent)) {
		//Running
		EnhancedInputComponent->BindAction(RunAction, ETriggerEvent::Started, this, &ATPSProjectCharacter::Run);
		//이동이 멈추면 달리기도 끝내야한다//그래서 MoveAction이 Completed 됐을 때 작동하는 함수로 등록
		EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Completed, this, &ATPSProjectCharacter::StopRun);
	}

}

//달리기 //bRunning값을 바꾸고 그에 따라 WalkSpeed 수정
//단 멈췄을때에 대한 대비가 필요
void ATPSProjectCharacter::Run() {

	//달리기 토글
	bRunning = !bRunning;

	//달리기 중
	if (bRunning) {
		//이동속도 2배
		GetCharacterMovement()->MaxWalkSpeed = 1000.f;
	}
	//걷는 중
	else {
		//원래 속도
		GetCharacterMovement()->MaxWalkSpeed = 500.f;
	}

}

// 이동 자체가 멈췄을 때
void ATPSProjectCharacter::StopRun() {
	bRunning = false;
	GetCharacterMovement()->MaxWalkSpeed = 500.f;
}

```

다음은 블루프린트에 IA_Run을 등록 해줍니다.

![3](/Assets/Images/Unreal/실습/TPSProject/2.추가기능/3.png)

다음 실제 동작이 원활히 되는지 확인해줍니다.
![4](/Assets/Images/Unreal/실습/TPSProject/2.추가기능/4.png)
