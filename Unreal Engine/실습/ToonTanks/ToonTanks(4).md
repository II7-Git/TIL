# ToonTanks(4)

## 게임 모드 설정

새로운 게임모드를 만든 뒤 프로젝트 세팅에서 설정해준다.

![12](/Assets/Images/Unreal/실습/ToonTanks/12.png)

## 플레이어 컨트롤러 설정

플레이어 컨트롤러를 만들어주어서 GameMode에 등록시키고 이 곳에서 게임의 컨트롤 권한 여부를 세팅해준다.

```C++
void AToonTanksPlayerController::SetPlayerEnabledState(bool bPlayerEnabled)
{
    if (bPlayerEnabled)
    {
        GetPawn()->EnableInput(this);
    }
    else
    {
        GetPawn()->DisableInput(this);
    }

    bShowMouseCursor = bPlayerEnabled;
}
```

## Timer Delegate

Timer함수에서 인풋을 받아들이는 함수는 주소를 적는 방식의 SetTimer를 쓸 수 없는데 이럴 때 사용하는 방식이 Timer Delegate라는 구조체를 이용해서 인풋이 있는 함수를 바인딩 하는 방식이다.

아래의 코드를 보면 인풋 파라미터가 필요한 함수의 콜백을 Timer Delegate를 통해서 구현한 것을 확인할 수 있다.

```C++
    FTimerHandle PlayerEnableTimerHandle;
    //함수의 주소와 넣을 인자값을 차례대로 넣어주면 된다.
    FTimerDelegate PlayerEnableTimerDelegate = FTimerDelegate::CreateUObject(
        ToonTanksPlayerController,
        &AToonTanksPlayerController::SetPlayerEnabledState,
        true);
    GetWorldTimerManager().SetTimer(PlayerEnableTimerHandle, PlayerEnableTimerDelegate, StartDelay, false);
```

## 게임 시작 위젯

### Blueprint Implementable Events

C++에서 작성하는 클래스이지만 블루프린트에서 구현이 되는 것을 뜻한다. 이를 호출하면 블루프린트 안에 함수가 동작하고 해당 이벤트에 연결 된 로직까지 호출이 된다.

이 함수로 만들고 싶다면 선언부에서 UFUNCTION(BlueprintImplementableEvent)을 적어서 블루프린트 함수로 만들어야한다.

Blueprint Implementable Events는 함수의 구현이 블루 프린트에서 이루어지기에 굳이 C++ 에서 함수의 바디를 구현할 필요는 없다.

```C++
//BluePrint에서 사용해야하므로 최소 protected
protected:
	UFUNCTION(BlueprintImplementableEvent)
	void StartGame();
```

이제 Blueprint 쪽에서 StartGame()을 구현하면 된다.

### WidgetBlueprint

화면 UI를 구성할 때 유용한 방식이다.Unity의 Canvas를 통한 UI구성과 비슷하다.

위젯 블루프린트를 만들어 원하는 UI를 구성 후 소환해주면 된다.

블루프린트 함수로 WBP_StartGame 블루프린트 위젯을 소환하여 뷰포트에 더해주는 모습
![15](/Assets/Images/Unreal/실습/ToonTanks/15.png)

### 위젯 제작

블루프린트 위젯의 디자이너 탭과 그래프 탭을 이용하여 원하는 UI를 구현해준다.

![16](/Assets/Images/Unreal/실습/ToonTanks/16.png)
![17](/Assets/Images/Unreal/실습/ToonTanks/17.png)

## 게임 오버와 승패 조건 구현

게임이 종료될 때 승패를 구분지어서 UI로 표시해주려고 한다.

1. 플레이어가 모든 타워를 파괴한다 -> 승리
2. 타워가 남아있는데 플레이어가 먼저 파괴된다 -> 패배

### 타워 개수 세기

먼저 타워 개수를 세야하기에 beginplay에서 Tower개수를 셀 수 있게끔 함수를 만들어준다.<br>이때 GetAllActorsOfClass()를 사용해 Tower의 포인터 리스트를 구해 개수를 세준다.

```C++
int32 AToonTanksGameMode::GetTargetTowerCount()
{
    TArray<AActor *> Towers;
    UGameplayStatics::GetAllActorsOfClass(this, ATower::StaticClass(), Towers);
    return Towers.Num();
}
```

### GameOver 함수 만들기

GameMode에 승패에 따라 다른 값을 출력하게끔 GameOver에 bool값이 인자로 들어가는 함수를 만든다. 이 때 블루프린트 함수로 만들어서 구현은 블루프린트에서 위젯을 소환하는 방식으로 한다.

```C++
UFUNCTION(BlueprintImplementableEvent)
	void GameOver(bool bWonGame);

```

### GameOver 호출

게임 오버 호출은 ActorDied에서 탱크가 죽었을 때와 타워가 죽어서 타워의 총 개수가 0 됐을 때 각각 호출한다

```C++
void AToonTanksGameMode::ActorDied(AActor *DeadActor)
{
    if (DeadActor == Tank)
    {
        Tank->HandleDestruction();
        if (ToonTanksPlayerController)
        {
            ToonTanksPlayerController->SetPlayerEnabledState(false);
        }
        GameOver(false);
    }
    else if (ATower *DestroyedTower = Cast<ATower>(DeadActor))
    {
        DestroyedTower->HandleDestruction();
        TargetTowers--;
        if (TargetTowers == 0)
        {
            GameOver(true);
        }
    }
}

```

### GameOver Blueprint 구현

아래와 같이 Won Game값에 따라 승패의 Text를 바꾸어 출력하게 하는 구현을 해준다.
![18](/Assets/Images/Unreal/실습/ToonTanks/18.png)

### 실행 확인

실제 플레이를 해보면 각각 조건에 따라 승패를 확인할 수 있었다.
![18](/Assets/Images/Unreal/실습/ToonTanks/19.png)
![18](/Assets/Images/Unreal/실습/ToonTanks/20.png)
