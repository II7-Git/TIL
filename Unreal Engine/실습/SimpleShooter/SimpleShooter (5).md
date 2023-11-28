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

## TActorRange<>()

특정 World 내에서 <>안에 넣은 객체들을 배열로 리턴해주는 메소드로 이를 통해서 남은 AI 적의 개수를 파악한다.

EndGame은 게임 종료 시에 남은 컨트롤러들을 전부 불러와 각 컨트롤러에게 GameHasEnded()를 사용해주는 메소드이다. 이 때 Player와 AI에게 각각 다른 승리값을 전달해주는 방식이다.

```C++
void AKillEmAllGameMode::EndGame(bool bIsPlayerWinner)
{
    // 월드 내에 있는 모든 ACotroller 객체 리턴
    for (AController *Controller : TActorRange<AController>(GetWorld()))
    {
        // 플레이어가 남으면 true ,AI만 남으면 false
        bool bIsWinner = Controller->IsPlayerController() == bIsPlayerWinner;
        Controller->GameHasEnded(Controller->GetPawn(), bIsWinner);
    }
}
```

## 최종 게임 종료 로직

위에서 만든 EndGame을 폰이 죽을 때마다 승패 여부가 만약 결정되면 그에 따라서 호출해주는 구조이다.

```C++
void AKillEmAllGameMode::PawnKilled(APawn *PawnKilled)
{
    Super::PawnKilled(PawnKilled);
    // 죽은 Pawn이 플레이어인 경우-> 게임 오버
    APlayerController *PlayerController = Cast<APlayerController>(PawnKilled->GetController());
    if (PlayerController != nullptr)
    {
        EndGame(false);
    }
    // AI 상대를 죽인 경우-> 남은 AI 개수를 세서 전부 사망시 게임 종료
    for (AShooterAIController *Controller : TActorRange<AShooterAIController>(GetWorld()))
    {

        if (!Controller->IsDead())
        {
            return;
        }
    }
    // 모든 AI를 없애서 승리함
    EndGame(true);
    return;
}
```

## 게임 종료 확인

실제 실행화면

패배 시
![30](/Assets/Images/Unreal/실습/SimpleShooter/30.png)

승리 시
![31](/Assets/Images/Unreal/실습/SimpleShooter/31.png)
