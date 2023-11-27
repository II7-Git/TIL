# SimpleShooter 5

## GameMode 제작

먼저 게임을 승패에 따른 처리를 관리할 게임모드를 만들어주어야 한다. KillEmAllGameMode를 만들어서 PawnKilled()라는 폰들이 죽었을 때 폰의 종류에 따른 킬 처리를 해주는 함수를 오버라이드 해주었다.

### 게임 승패에 따른 GameHasEnded와 PlayerController 처리

게임이 패배했을 때를 먼저 구현하면 죽은 폰이 플레이어일 경우 GetController를 하면 APlayerController가 return 되기에 이는 플레이어의 사망으로 판단해도 된다. 따라서 이를 기준으로 사망했을 시에는 APlayerController의 GameHasEnded를 호출해준다.

```C++
void AKillEmAllGameMode::PawnKilled(APawn *PawnKilled)
{
    Super::PawnKilled(PawnKilled);

    APlayerController *PlayerController = Cast<APlayerController>(PawnKilled->GetController());
    if (PlayerController != nullptr)
    {
        PlayerController->GameHasEnded(nullptr, false);
    }
}

```

GameHasEnded 같은 경우는 PlayerController의 가상 함수이므로 원하는 대로 커스터마이징이 가능하다. 이번 경우는 지는 경우에 질 때 사용할 수 있는 UI를 소환하는 구조로 구성했다. UI 같은 경우는 위젯 블루프린트로 간단히 생성하였다.

```C++
void AShooterPlayerController::GameHasEnded(AActor *EndGameFocus, bool bIsWinner)
{
    Super::GameHasEnded(EndGameFocus, bIsWinner);

    UUserWidget *LoseScreen = CreateWidget(this, LoseScreenClass);
    if (LoseScreen != nullptr)
    {
        LoseScreen->AddToViewport();
    }
    GetWorldTimerManager().SetTimer(RestartTimer, this, &APlayerController::RestartLevel, RestartDelay);
}
```

실제 실행화면
![30](/Assets/Images/Unreal/실습/SimpleShooter/30.png)
