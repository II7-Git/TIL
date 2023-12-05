# SimpleShooter 6

## 사운드

### 사운드 큐

소리를 다양하게 해줄 수 있는 언리얼에서 제공되는 기능으로 애니메이션 설정처럼 블루프린트를 통해서 설정이 가능하다.

아래처럼 랜덤한 소리를 재생하고 모듈레이터로 소리의 크기나 피치를 변환시켜줌으로써 다채로운 사운드를 재생 가능하게 했다.

![32](/Assets/Images/Unreal/실습/SimpleShooter/32.png)

### 사운드 공간화

사운드가 거리나 방향에 따라 소리가 다르게 들리는 것을 구현하는 기능이다.

사운드 큐에서 사운드 어테뉴에이션을 설정해 줄 수 있는데 이를 통해서 방향감을 구현한다. 이를 따로 분리하여 구현하여 여러 사운드 큐에서 같은 설정을 사용하는 것도 가능하다.
![34](/Assets/Images/Unreal/실습/SimpleShooter/34.png)
![33](/Assets/Images/Unreal/실습/SimpleShooter/33.png)

### 소리 발생

총기 발사 할 때랑 맞는 곳 위치에 Effect를 설정해놓았는데 그 위치에 사운드도 넣어준다.

```C++
// 총구에서 발사될 때 소리 부착
UGameplayStatics::SpawnEmitterAttached(MuzzleFlash, Mesh, TEXT("MuzzleFlashSocket"));
UGameplayStatics::SpawnSoundAttached(MuzzleSound, Mesh, TEXT("MuzzleFlashSocket"));

// 발사된 탄 맞는 위치에 소리 생성
UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ImpactEffect, Hit.Location, ShotDirection.Rotation());
UGameplayStatics::SpawnSoundAtLocation(GetWorld(), ImpactSound, Hit.Location, ShotDirection.Rotation());
```

## HUD

크로스헤어와 체력 정보를 나타내 줄 UI를 제작한다.<br>
이렇게 만든 HUD는 게임 시작할 때 나타나고 플레이어가 죽으면 사라져야하기에 BeginPlay()에서 뷰포트에 붙히고 죽을 때 제거한다.

```C++
//뷰포트에 HUD UI 부착
void AShooterPlayerController::BeginPlay()
{
    Super::BeginPlay();

    HUD = CreateWidget(this, HUDClass);
    if (HUD != nullptr)
    {
        HUD->AddToViewport();
    }
}

//기존 GameHasEnded에서 HUD 제거 부분 추가
void AShooterPlayerController::GameHasEnded(AActor *EndGameFocus, bool bIsWinner)
{
    Super::GameHasEnded(EndGameFocus, bIsWinner);

    HUD->RemoveFromViewport();
}

```

### 크로스헤어

먼저 크로스 헤어는 간단히 UI 중앙에 + 자를 사용해서 제작했다.

### HP

HP는 UI의 프로그레스 바의 기능과 동일하다 프로그레스 바는 0~1사이에 값을 가지고 바 형태로 채워주기에 이를 활용한다.

프로그레스 바의 채우기 오파시티 채우기에서 바인드를 통해 해당 UI를 소유한 컨트롤러의 폰의 정보에서 체력/최대 체력을 하여 0~1의 값을 갱신하게 한다.<br>

바인딩 옵션<br>
![37](/Assets/Images/Unreal/실습/SimpleShooter/37.png)

ShooterCharacter는 블루프린트 순수 함수 GetHealth를 만들고 이를 통해 오파시티 채우기 값 구현<br>
![36](/Assets/Images/Unreal/실습/SimpleShooter/36.png)

실제 인게임 적용 모습<br>
![35](/Assets/Images/Unreal/실습/SimpleShooter/35.png)

## 에임 오프셋

플레이어가 에임의 위치를 수정하면 총구도 같이 따라 움직여야 좀 더 자연스러운 모션이 된다. 이를 위해 존재하는 것이 에임 오프셋 애니메이션인데 에디티브 애니메이션 속성이므로 기존의 걷는 애니메이션의 추가하면 된다.

기존 애니메이션의 에임 오프셋 애니메이션 추가<br>
![40](/Assets/Images/Unreal/실습/SimpleShooter/40.png)

이때 에임 오프셋에 설정할 Pitch와 Yaw 값은 다음 사진처럼 가져온다.<br>
![39](/Assets/Images/Unreal/실습/SimpleShooter/39.png)

이는 컨트롤러의 Rotation - 플레이어 Rotation 을 하여 플레이어 액터 기준에서 컨트롤러와의 Rotation 차이 값을 계산하여 정확한 위치의 오프셋을 계산하게 해준다.

구현 모습
![38](/Assets/Images/Unreal/실습/SimpleShooter/38.png)

## 애니메이션 스테이트

애니메이션 스테이트는 여러 변수 상태에 따라서 애니메이션간의 변화를 적용시킬 때 유용하다. 또 스테이트 안에 스테이트를 만들어 계층화 구조를 만들기도 편하다.

예를 들어 점프하는 동안에는 걷는 동작을 취하면 안되니 Character에 있는 캐릭터 무브먼트에서 IsFalling 값을 가져와서 이 값의 여부에 따른 스테이트 머신을 구축한다.<br>
이를 통해서 애니메이션이 자연스럽게 전환되는 것을 구현한다.<br>
![42](/Assets/Images/Unreal/실습/SimpleShooter/42.png)

또 슈터 캐릭터는 큰 틀로 죽은 상태의 애니메이션과 산 상태의 동작 애니메이션이 있으므로 이를 각각 스테이트화 시켜 IsDead 여부의 따라 전환하는 계층화 구조를 취할 수 있다.<br>

가장 큰 틀의 애님 그래프<br>
![43](/Assets/Images/Unreal/실습/SimpleShooter/43.png)

Death 스테이트 머신 모습<br>
![44](/Assets/Images/Unreal/실습/SimpleShooter/44.png)

이 중 Alive에는 기존에 구현해두었던 동작 애니메이션들을 Dead에는 사망 애니메이션을 구현하고 IsDead 변수에 따라 전환한다.

# 프로젝트 마무리

이것으로 프로젝트의 모든 기능의 구현이 끝났다. 추가적으로 배경 사운드와 Intro 사운드 등 엠비언트 사운드를 통한 작업을 해주고 적들의 적절한 배치를 통해서 레벨 디자인을 하고 프로젝트를 마무리 지었다.
