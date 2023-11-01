# SimpleShooter 1

심플 슈터는 TPS방식으로 제작되는 게임으로 적을 전부 파괴하면 끝나는 게임이다.

기본 제공되는 무료 에셋들을 가지고 작업을 진행한다.
![1](/Assets/Images/Unreal/실습/SimpleShooter/1.png)

## 캐릭터 만들기

먼저 캐릭터 C++클래스를 만들고 이를 바탕으로 파생 블루 프린트를 만들고 디폴트 폰으로 지정해준다.

### 움직임 구현

플레이어의 캐릭터는 Character 클래스를 상속받았기 때문에 기본 움직임 함수는 다 구현 되어 있기에 이를 축바인딩에서 연결 시켜주는 작업으로 간단하게 wasd 이동과 마우스 카메라 회전을 구현해놓았다.

이때 컨트롤러 이동에서는 DeltaTime을 사용해서 이동값이 프레임 변동에서도 일정할 수 있게끔 조절해준다.

```C++
// Called to bind functionality to input
void AShooterCharacter::SetupPlayerInputComponent(UInputComponent *PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

	PlayerInputComponent->BindAxis(TEXT("MoveForward"), this, &AShooterCharacter::MoveForward);
	PlayerInputComponent->BindAxis(TEXT("MoveRight"), this, &AShooterCharacter::MoveRight);
	PlayerInputComponent->BindAxis(TEXT("LookUp"), this, &APawn::AddControllerPitchInput);
	PlayerInputComponent->BindAxis(TEXT("LookRight"), this, &APawn::AddControllerYawInput);
	PlayerInputComponent->BindAction(TEXT("Jump"), EInputEvent::IE_Pressed, this, &ACharacter::Jump);

	// 컨트롤러용
	PlayerInputComponent->BindAxis(TEXT("LookUpRate"), this, &AShooterCharacter::LookUpRate);
	PlayerInputComponent->BindAxis(TEXT("LookRightRate"), this, &AShooterCharacter::LookRightRate);
}

void AShooterCharacter::MoveForward(float AxisValue)
{
	AddMovementInput(GetActorForwardVector() * AxisValue);
}

void AShooterCharacter::MoveRight(float AxisValue)
{
	AddMovementInput(GetActorRightVector() * AxisValue);
}

//Controller용으로 DeltaTime을 사용해서 속도를 일정하게 조정해준다.
void AShooterCharacter::LookUpRate(float AxisValue)
{
	AddControllerPitchInput(AxisValue * RotationRate * GetWorld()->GetDeltaSeconds());
}

void AShooterCharacter::LookRightRate(float AxisValue)
{
	AddControllerYawInput(AxisValue * RotationRate * GetWorld()->GetDeltaSeconds());
}
```

### 카메라 배치

3인칭 시점에 맞게 오른 어깨 뒤에 스프링 암과 카메라를 달고 앵글을 조정해준다.
![2](/Assets/Images/Unreal/실습/SimpleShooter/2.png)
