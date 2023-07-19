# FPS 프로젝트 (3)

FPS에서 빠질 수 없는 발사체 구현을 해보겠습니다.

## 1.발사체 추가

먼저 발사를 하기 위한 매핑을 진행합니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/1.png)

다음은 FPSProjectile 클래스를 추가해줍니다. 부모 클래스는 Actor를 상속받습니다.

### FPSProjectile.h

header에 필요하다면 추가해줍니다

```C++
#include "Components/SphereComponent.h"
```

public에 추가해줍니다.

```C++

    // 구체 콜리전 컴포넌트입니다.
	UPROPERTY(VisibleDefaultsOnly, Category = Projectile)
		USphereComponent* CollisionComponent;
```

### FPSProjectile.cpp

```C++
    // 구체를 단순 콜리전 표현으로 사용합니다.
	CollisionComponent = CreateDefaultSubobject<USphereComponent>(TEXT("SphereComponent"));

	// 구체의 콜리전 반경을 설정합니다.
	CollisionComponent->InitSphereRadius(15.0f);
	// 루트 컴포넌트를 콜리전 컴포넌트가 되도록 설정합니다.
	RootComponent = CollisionComponent;
```

---

다음은 UProjectileMovementComponent를 추가해줍니다.

### FPSProjectile.h

header에 필요하다면 추가해줍니다

```C++
#include "Engine/Classes/GameFramework/ProjectileMovementComponent.h"
```

public에 추가해줍니다.

```C++

   // 프로젝타일 무브먼트 컴포넌트
    UPROPERTY(VisibleAnywhere, Category = Movement)
    UProjectileMovementComponent* ProjectileMovementComponent;
```

### FPSProjectile.cpp

```C++
   // ProjectileMovementComponent 를 사용하여 이 발사체의 운동을 관장합니다.
    ProjectileMovementComponent = CreateDefaultSubobject<UProjectileMovementComponent>(TEXT("ProjectileMovementComponent"));
    ProjectileMovementComponent->SetUpdatedComponent(CollisionComponent);
    ProjectileMovementComponent->InitialSpeed = 3000.0f;
    ProjectileMovementComponent->MaxSpeed = 3000.0f;
    ProjectileMovementComponent->bRotationFollowsVelocity = true;
    ProjectileMovementComponent->bShouldBounce = true;
    ProjectileMovementComponent->Bounciness = 0.3f;
```

---

### 발사체의 초기 속도를 설정하겠습니다.

### FPSProjectile.h

```C++
// 발사체의 속도를 발사 방향으로 초기화시킵니다.
void FireInDirection(const FVector& ShootDirection);
```

### FPSProjectile.cpp

```C++
   // 프로젝타일의 속도를 발사 방향으로 초기화시키는 함수입니다.
    void AFPSProjectile::FireInDirection(const FVector& ShootDirection)
    {
        ProjectileMovementComponent->Velocity = ShootDirection * ProjectileMovementComponent->InitialSpeed;
    }
```

---

### 발사체 액션 바인딩하겠습니다.

### FPSCharacter.h

```C++
// 발사 처리
UFUNCTION()
void Fire();
```

### FPSCharacter.cpp

SetupPlayerInputComponent 에 다음 바인딩을 추가합니다.

```C++
   PlayerInputComponent->BindAction("Fire", IE_Pressed, this, &AFPSCharacter::Fire);
```

---

### 발사체 스폰 위치 설정

발사체가 어디서 생성될지 정해보겠습니다.

### FPSCharacter.h

```C++
// 카메라 위치에서의 총구 오프셋
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Gameplay)
FVector MuzzleOffset;

// 스폰시킬 프로젝타일 클래스
UPROPERTY(EditDefaultsOnly, Category = Projectile)
TSubclassOf<class AFPSProjectile> ProjectileClass;
```

## 2.발사 구현

### FPSCharacter.cpp에 임포트 해줍니다.

```C++
#include "FPSProjectile.h"
```

### FPSCharacter.cpp

fire 함수를 구현해줍니다.<br>
주의할 점은 언리얼 5로 와서는 Instigator가 아닌 GetInstigator()로 값을 얻어와야합니다.

```C++
void AFPSCharacter::Fire()
{
	// 프로젝타일 발사를 시도합니다.
	if (ProjectileClass)
	{
		// 카메라 트랜스폼을 구합니다.
		FVector CameraLocation;
		FRotator CameraRotation;
		GetActorEyesViewPoint(CameraLocation, CameraRotation);

		// MuzzleOffset 을 카메라 스페이스에서 월드 스페이스로 변환합니다.
		FVector MuzzleLocation = CameraLocation + FTransform(CameraRotation).TransformVector(MuzzleOffset);
		FRotator MuzzleRotation = CameraRotation;
		// 조준을 약간 윗쪽으로 올려줍니다.
		MuzzleRotation.Pitch += 10.0f;
		UWorld* World = GetWorld();
		if (World)
		{
			FActorSpawnParameters SpawnParams;
			SpawnParams.Owner = this;
			//언리얼 5 엔진에서 대체되었습니다. Instigator -> GetInstigator()
			SpawnParams.Instigator = GetInstigator();
			// 총구 위치에 발사체를 스폰시킵니다.
			AFPSProjectile* Projectile = World->SpawnActor<AFPSProjectile>(ProjectileClass, MuzzleLocation, MuzzleRotation, SpawnParams);
			if (Projectile)
			{
				// 발사 방향을 알아냅니다.
				FVector LaunchDirection = MuzzleRotation.Vector();
				Projectile->FireInDirection(LaunchDirection);
			}
		}
	}
}
```

---

[프로젝타일 메시 LINK](https://docs.unrealengine.com/4.26/Attachments/ProgrammingAndScripting/ProgrammingWithCPP/CPPTutorials/FirstPersonShooter/3/2/Sphere.zip)

먼저 위 링크에서 다운받은 메시를 import 해줍니다.

다음 블루프린트 신규 추가 해주어 FPSProjectile을 상속받은 블루프린트를 생성합니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/2.png)

다음은 이 블루프린트를 열어 메시를 할당해줍니다.<br>
설정 값은 아래 이미지를 따라해주시면 됩니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/3.png)

---

다음은 캐릭터 블루프린터를 열어줍니다.<br>
그 뒤 detail 탭에서 projectile을 검색하여 방금 생성한 프로젝타일 블루프린트를 바인딩해줍니다.<br>
추가로 muzzle offset도 변경해줍니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/4.png)
![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/5.png)

다 완료되셨으면 Compile후 실행하여 발사체를 확인하시면 됩니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/6.png)

## 3.발사체 콜리전 및 수명 구성

### FPSProjectile.cpp

생성자에 추가합니다.

```C++
// 3 초 후 죽습니다.
InitialLifeSpan = 3.0f;
```

---

프로젝트 세팅 -> 콜리전

이 곳에서 콜리전 세팅을 진행합니다.

먼저 오브젝트 채널 추가해줍니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/7.png)

다음은 프리셋을 추가해줍니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/8.png)

---

새 콜리전을 세팅해보겠습니다.

### FPSProjectile.cpp

생성자에 추가해줍니다.

```C++
CollisionComponent->BodyInstance.SetCollisionProfileName(TEXT("Projectile"));
```

컴파일해서 확인 후 다음 스텝으로 넘어가겠습니다.

## 4.월드와 상호작용하는 프로젝타일

프로젝타일이 콜리전에 반응하도록 작업을 진행하겠습니다.

### FPSProjectile.h

```C++
	// 프로젝타일이 무언가에 맞으면 호출되는 함수입니다.
	UFUNCTION()
		void OnHit(UPrimitiveComponent* HitComponent, AActor* OtherActor, UPrimitiveComponent* OtherComponent, FVector NormalImpulse, const FHitResult& Hit);
```

### FPSProjectile.cpp

```C++
// 프로젝타일에 무언가 맞으면 호출되는 함수입니다.
void AFPSProjectile::OnHit(UPrimitiveComponent* HitComponent, AActor* OtherActor, UPrimitiveComponent* OtherComponent, FVector NormalImpulse, const FHitResult& Hit)
{
    if (OtherActor != this && OtherComponent->IsSimulatingPhysics())
    {
        OtherComponent->AddImpulseAtLocation(ProjectileMovementComponent->Velocity * 100.0f, Hit.ImpactPoint);
    }
}
```

생성자에 추가해줍니다.

```C++
CollisionComponent->OnComponentHit.AddDynamic(this, &AFPSProjectile::OnHit);
```

---

프로젝타일 블루프린트에 들어가서 생성한 콜리전들을 설정해줍니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/9.png)

---

다음은 테스트를 위해 맵에 오브젝트를 생성 후 피직스 탭에 피직스 시뮬레이트를 켜줍니다.

그 뒤 컴파일하여 테스트해보겠습니다.

## 5.뷰포트에 조준선 추가

화면에 조준선을 추가해보겠습니다.

[조준선 이미지 LINK](https://docs.unrealengine.com/4.26/Attachments/ProgrammingAndScripting/ProgrammingWithCPP/CPPTutorials/FirstPersonShooter/3/5/Crosshair_fps_tutorial.zip)

다운 후에 임포트를 진행해주시면 됩니다.

다음은 HUD 클래스를 상속받은 FPSHUD 클래스를 생성해줍니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/10.png)

### FPSHUD.h

```C++
protected:
    // 화면 중앙에 그려질 것입니다.
    UPROPERTY(EditDefaultsOnly)
    UTexture2D* CrosshairTexture;
public:
    // HUD 에 대한 주요 드로 콜입니다.
    virtual void DrawHUD() override;
```

### FPSHUD.cpp

```C++
void AFPSHUD::DrawHUD()
{
    Super::DrawHUD();

    if (CrosshairTexture)
    {
        // 캔버스 중심을 찾습니다.
        FVector2D Center(Canvas->ClipX * 0.5f, Canvas->ClipY * 0.5f);

        // 텍스처 중심이 캔버스 중심에 맞도록 텍스처의 크기 절반 만큼 오프셋을 줍니다.
        FVector2D CrossHairDrawPosition(Center.X - (CrosshairTexture->GetSurfaceWidth() * 0.5f), Center.Y - (CrosshairTexture->GetSurfaceHeight() * 0.5f));

        // 중심점에 조준선을 그립니다.
        FCanvasTileItem TileItem(CrossHairDrawPosition, CrosshairTexture->Resource, FLinearColor::White);
        TileItem.BlendMode = SE_BLEND_Translucent;
        Canvas->DrawItem(TileItem);
    }
}
```

---

컴파일을 한 뒤 블루프린트로 FPSHUD를 확장시킵니다.

그 뒤 해당 블루프린트를 해당 프로젝트의 기본 HUD로 설정합니다.

![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/11.png)

---

마지막으로 해당 블루프린트를 열어 디테일 탭에서 FPSHUD를 찾아서 다운받은 이미지를 할당해줍니다.
![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/12.png)

### 최종 구현

![TEXT](/Assets/Images/Unreal/실습/FPSProject/3/13.png)

## 끝 마치며

이번은 공을 생성해서 날리는 것을 구현해보았는데 다음에는 총알을 구현해봐서 데미지를 추가해 좀 더 게임적인 요소를 넣어서 진행해보고 싶다는 생각이 들었습니다.

다음은 마지막인 캐릭터 애니메이션 처리를 진행해보겠습니다.
