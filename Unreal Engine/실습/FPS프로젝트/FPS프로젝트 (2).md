# FPS프로젝트 (2)

전에 이어서 이번엔 캐릭터의 구현을 진행해보겠습니다. 해당 스텝은 본격적인 언리얼에 기능들을 사용하여 플레이어의 캐릭터를 구현하는 단계입니다.

# 캐릭터 임포트

# 1. 새 캐릭터 클래스 생성

먼저 캐릭터 C++ 클래스를 생성해주어야 하는데 Character를 상속받아 생성해줍니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/1.png)

다음은 해당 클래스를 실제 사용하기 위해 블루프린트로 확장을 합니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/2.png)

이제 해당 블루프린트를 실제 프로젝트에서 사용될 플레이어로 등록해야 합니다. 이러한 설정들은 대게 '프로젝트 세팅' 에서 진행하게 됩니다.<br>
프로젝트 세팅-> Maps & Modes -> Default Pawn Class <br>
로 가셔서 블루프린트를 등록해줍니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/3.png)

---

# 2. 축 매핑 구성

이번은 캐릭터의 WSAD 동작을 처리해보겠습니다.<br>
프로젝트 세팅-> 입력(Input)<br>
탭으로 들어가서 축 매핑에 다음 이미지처럼 추가해줍니다. 이를 통해 실제 키보드의 입력이 해당 이름으로 매핑된다는 것을 뜻하게 됩니다. 이를 통해 C++에서 동작을 처리하게 됩니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/4.png)

# 3. 캐릭터 동작 구현

축 매핑을 실제 동작으로 구현하게 되는데 FPSCharacter에서 해당 작업들이 진행됩니다.

## FPSCharacter.h

```C++
public:
// 전후 이동 처리
	UFUNCTION()
		void MoveForward(float Value);

	// 좌우 이동 처리
	UFUNCTION()
		void MoveRight(float Value);
```

FPSCharacter.cpp 에서는 이에 대한 구현을 진행합니다.<br>
캐릭터에 프로젝트 세팅에 적어놓은 매핑들을 바인딩하는 것은 <br>
AFPSCharacter::SetupPlayerInputComponent(UInputComponent\* PlayerInputComponent)<br>
에서 하게 됩니다.

먼저 헤더에서 선언한 함수들을 구현한 뒤 해당 함수에 바인딩을 해주시면 됩니다.

## FPSCharacter.cpp

```C++
// Called to bind functionality to input
void AFPSCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

	// "movement" 바인딩 구성
	PlayerInputComponent->BindAxis("MoveForward", this, &AFPSCharacter::MoveForward);
	PlayerInputComponent->BindAxis("MoveRight", this, &AFPSCharacter::MoveRight);
}

void AFPSCharacter::MoveForward(float Value)
{
	// 어느 쪽이 전방인지 알아내어, 플레이어가 그 방향으로 이동하고자 한다고 기록합니다.
	FVector Direction = FRotationMatrix(Controller->GetControlRotation()).GetScaledAxis(EAxis::X);
	AddMovementInput(Direction, Value);
}

void AFPSCharacter::MoveRight(float Value)
{
	// 어느 쪽이 오른쪽인지 알아내어, 플레이어가 그 방향으로 이동하고자 한다고 기록합니다.
	FVector Direction = FRotationMatrix(Controller->GetControlRotation()).GetScaledAxis(EAxis::Y);
	AddMovementInput(Direction, Value);
}
```

작성을 마친 뒤 에디터에 콘솔창에 liveCoding.compile 을 입력하여 Compile 후 동작을 확인합니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/5.png)

---

### 4. 마우스 카메라 컨트롤 구현

기본적인 흐름은 위 WSAD 구현과 비슷하게 진행됩니다.

먼저 마우스 동작을 매핑해줍니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/6.png)

마찬가지로 FPSCharacter에 작성합니다.

## FPSCharacter.cpp

```C++
// Called to bind functionality to input
void AFPSCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

	// "look" 바인딩 구성
	// 마우스 감도나 축 반전 같은 추가 처리를 해주려면 함수를 추가해야 한다. 그렇지만 그냥 매핑 시키려면 Character에서 제공되는 기본 함수에 바인딩 시켜도 무방하다.
	PlayerInputComponent->BindAxis("Turn", this, &AFPSCharacter::AddControllerYawInput);
	PlayerInputComponent->BindAxis("LookUp", this, &AFPSCharacter::AddControllerPitchInput);

}
```

상속해준 Character에서 기본적인 마우스 화면 동작 처리를 해주는 함수 AddControllerYawInput,AddControllerPitchInput 이 존재하기에 별도의 기능이 필요하지 않다면 그냥 해당 기능과 매핑을 바인딩만 해주어도 동작합니다.

마찬가지로 컴파일 하면 동작을 확인해볼 수 있습니다.

---

# 5. 캐릭터 점프 구현

캐릭터의 점프 구현입니다. 기본 Character 클래스에서 점프 기능을 지원하기에 이에 관한 변수 'bPressedJump' 를 관리해주기만 하면 점프 기능이 구현됩니다.

그 밖은 위에 매핑 과정을 그대로 따라갑니다.

점프는 버튼 동작의 기능이기에 액션 매핑에 추가해줍니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/13.png)

## FPSCharacter.h

public에 추가해줍니다.

```C++
//character.h를 보게 되면 bPressedJump 변수가 존재해서 이에 대해서 true false 처리만 해주면 되기에 아래 함수들을 통해서 처리만 해주면 된다.
	// 키를 누르면 점프 플래그를 설정합니다.
	UFUNCTION()
		void StartJump();

	// 키를 떼면 점프 플래그를 지웁니다.
	UFUNCTION()
		void StopJump();

```

## FPSCharacter.cpp

```C++
void AFPSCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);
	// "jump" 바인딩 구성
	PlayerInputComponent->BindAction("Jump",IE_Pressed ,this, &AFPSCharacter::StartJump);//눌렀을 때 동작
	PlayerInputComponent->BindAction("Jump", IE_Released,this, &AFPSCharacter::StopJump);//뗏을 때 동작

}

//점프는 캐릭터에서 제공되는 bPressedJump 변수를 다루는 것으로 간단히 처리
void AFPSCharacter::StartJump() {
	bPressedJump = true;
}

void AFPSCharacter::StopJump() {
	bPressedJump = false;
}
```

실행을 확인해보면 됩니다.

---

# 6. 캐릭터에 메시 추가

먼저 샘플 메시가 필요한데 아래 링크를 통해 다운받아 줍니다.<br>
[샘플 메시 Link](https://docs.unrealengine.com/4.26/Attachments/ProgrammingAndScripting/ProgrammingWithCPP/CPPTutorials/FirstPersonShooter/2/6/GenericMale.zip)

해당 파일 압축을 풀면 fbx 파일이 나오게 되는데 Contents 폴더에서 오른쪽 마우스 클릭 후 '/Game에 임포트' 를 클릭하여 해당 FBX를 임포트해옵니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/8.png)

다음은 블루프린트 FPSCharacter를 열어서 컴포넌트 탭에 Mesh 컴포넌트를 클릭 후 Details 탭에서 Mesh 부분에 방금 임포트 해온 FBX를 등록해주고 위치를 아래 사진과 같이 캡슐 콜리전 안에 들어갈 수 있도록 배치합니다.

그러면 아래 사진처럼 캐릭터 메시가 적용이 된 것을 확인할 수 있습니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/9.png)

컴파일 후 f8을 눌러 다른 시점으로 확인하면 아래 사진과 같이 캐릭터가 존재하는 것을 볼 수 있습니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/10.png)

---

# 7. 카메라 뷰 변경

카메라를 보다 일인칭 시점에 맞는 위치로 변경하기 위한 작업을 진행해 보겠습니다. 먼저 카메라 컴포넌트를 생성하고 해당 컴포넌트를 캐릭터 캡슐 컴포넌트에 붙이는 작업을 진행하고 위치를 조정하여 아래 이미지 같은 1인칭 카메라 위치를 생성합니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/11.png)

## FPSCharacter.h

주의할 점으로 UCamraComponent나 CapsuleComponent가 안잡히면 해당 내용을 import해와야 합니다.<br>
이 때 generated.h보다 위로 오도록 해야합니다.

```C++

//필요 컴포넌트들이 없다고 뜨면 명시해주어야한다
#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "Engine/Classes/Camera/CameraComponent.h"
#include "Components/CapsuleComponent.h"
#include "FPSCharacter.generated.h"


	// FPS 카메라
	//UCameraComponent가 없다면 해당 class를 임포트 해와야합니다!
	UPROPERTY(VisibleAnywhere)
		UCameraComponent* FPSCameraComponent;

```

## FPSCharacter.cpp

생성자에 추가해줍니다.

```C++
// 없다면 임포트를 해와야합니다
	// 일인칭 카메라 컴포넌트를 생성합니다.
	FPSCameraComponent = CreateDefaultSubobject<UCameraComponent>(TEXT("FirstPersonCamera"));
	// 카메라 컴포넌트를 캡슐 컴포넌트에 붙입니다
	//캡슐 컴포넌트 에러가 난다면 capsuleComponent.h룰 import 해와야합니다. ->#include "Components/CapsuleComponent.h"
	FPSCameraComponent->SetupAttachment(GetCapsuleComponent());

	// 카메라 위치를 눈 살짝 위로 잡습니다.
	FPSCameraComponent->SetRelativeLocation(FVector(0.0f, 0.0f, 50.0f + BaseEyeHeight));
	// 폰의 로테이션 제어를 허가합니다.
	FPSCameraComponent->bUsePawnControlRotation = true;

```

compile하여 확인해줍니다.

---

# 8.일인칭 메시 추가

일인칭 게임을 해보면 플레이어의 팔이 보이게됩니다. 이 경우에 맞춰 플레이어 팔 메시를 추가해보겠습니다.

## FPSCharacter.h

```C++
// 일인칭 메시 (팔), 소유 플레이어에게만 보입니다.
UPROPERTY(VisibleDefaultsOnly, Category = Mesh)
USkeletalMeshComponent* FPSMesh;
```

## FPSCharacter.cpp

생성자에 추가합니다.

```C++
// 일인칭 메시 컴포넌트를 생성합니다.
	FPSMesh = CreateDefaultSubobject<USkeletalMeshComponent>(TEXT("FirstPersonMesh"));
	// 이 메시는 소유 플레이어게만 보입니다.
	FPSMesh->SetOnlyOwnerSee(true);
	// FPS 카메라에 FPS 메시를 붙입니다.
	FPSMesh->SetupAttachment(FPSCameraComponent);
	// 일부 환경 섀도잉을 꺼서 메시가 하나처럼 보이는 느낌을 보존합니다.
	FPSMesh->bCastDynamicShadow = false;
	FPSMesh->CastShadow = false;

	// 소유 플레이어는 일반 (삼인칭) 바디 메시를 보지 못합니다.
	GetMesh()->SetOwnerNoSee(true);
```

[팔 메시 Link](https://docs.unrealengine.com/4.26/Attachments/ProgrammingAndScripting/ProgrammingWithCPP/CPPTutorials/FirstPersonShooter/2/8/HeroFPP.zip)

위 링크에서 팔 메시를 다운 후 임포트 해주고 등록하여 줍니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/11.png)

그 뒤 컴파일 하여 확인합니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/2/12.png)

### 끝 마치며

캐릭터를 제로부터 구현해보는 것은 첫 실습인데 언리얼 엔진의 에디터와 블루프린트와 C++의 연계성과 그에 따른 작업적 이점을 볼 수 있었던 것 같아서 인상깊었습니다. 또 Charater Class에서 기본 제공되는 기능이나 위에 말했던 세가지의 기본 제공 기능들이 훌륭해서 작업의 편의성이 많이 오르겠구나 생각이 들었습니다.

반면 아쉬운 점은 자잘한 오류들 때문에 기능이 정상적으로 동작하려면 추가 수정이 필요하다는 점 등이 아쉬웠습니다.

개인적으로 인상깊었던 부분들을 적자면

```
- 언리얼 엔진에서 헤더 파일에 다른 헤더를 include 할 때는 반드시 generated.h 위에 include 해야한다

  > 그 이유로는 generated.h는 컴파일 전에 생성되기에 이에 필요한 헤더들은 전부 이 위에 작성되어있어야 한다고 한다.<br>
  > 또 이렇게 작성하였는데 헤더 파일에서 GENRATED_BODY() 가 에러 표시가 나있는 경우가 있는데 단순히 보이는 것만 그러니 컴파일 하면 별 문제 없다.

- 기본적으로 상속 받은 클래스에 포함되어 정의되어 있는 클래스나 함수가 없다면?<br>
  그 클래스가 포함된 헤더 파일을 include하여 명시해주어야한다- 꾸준히 있어왔던 오류라곤 하지만 항상 이에 대한 걸 주의깊게 봐야겠다고 생각했다.<br>
  만약 어디에 포함된 클래스인지 못찾겠다면 공식 문서 API를 확인하면 좋다.

- 언리얼 엔진은 C++을 바탕으로 블루프린트를 만들어 에디터 위에서 작업하는 복합적인 작업물이 많다.<br>
  실습하면서 인상 깊었던 것은 C++에서 모든 걸 구현하고 단순 매핑이 아니라 C++에선 기능의 처리를 블루프린트에서는 실제 레벨에서 보일 오브젝트의 구현과 매핑을 에디터에서는 게임에 설정에 관한 세팅을 진행하는 복합적인 작업이 기본적인 프로젝트의 동작 방식이었다는 점이었다.
```

등이 있습니다.

다음은 발사체 구현을 진행하겠습니다.
