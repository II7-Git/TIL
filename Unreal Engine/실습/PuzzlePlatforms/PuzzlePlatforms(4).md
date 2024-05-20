# PuzzlePlatforms(4)

## Lobby 생성

세션을 통해서 방을 만들었다면 이제 해당 방에 사람이 일정 수 이상 모여야지 시작하는 구조를 구성하려고 했다.

그래서 로비 레벨을 생성하고 해당 레벨에 사람이 일정 수 이상 모이면 게임 레벨로 이동하여 게임을 시작하는 구조를 만들었다.

이 때 세션에 모인 사람 수를 관리하여야 하는데 이 기능을 GameMode에서 관리하게 하였다. 그래서 LobbyGameMode를 만들고 이곳에서 세션에 들어온 사람이 있을 때 발동되는 비동기 함수인 PostLogin(APlayerController \*NewPlayer)을 override하여서 들어온 사람 수를 관리하고 3명 이상이면 게임 시작으로 이동하게 구성했다.

```C++
void ALobbyGameMode::PostLogin(APlayerController *NewPlayer)
{
    Super::PostLogin(NewPlayer);
    NumberOfPlayers++;

    if (NumberOfPlayers >= 3)
    {
        UWorld *World = GetWorld();

        if (World == nullptr)
            return;

        World->ServerTravel("/Game/Maps/Game?listen");
    }
}
```

그리고 방에서 나간 사람은 Logout(AController \*Exiting)을 override해서 관리해주었다.

```C++
void ALobbyGameMode::Logout(AController *Exiting)
{
    Super::Logout(Exiting);
    NumberOfPlayers--;
}
```

### Seamless Travel

심리스 이동에 대한 설명과 설정 방법은 아래 링크에 정리해두었다.

#### [Seamless Travel 정리](/Unreal%20Engine/이론%20및%20정리/멀티플레이/Seamless%20Travel.md)

실제 코드에서는 아래와 같이 구성했다.<br>
bUseSeamlessTravel을 켜주어서 Seamless 이동방식을 구현했다.

```C++
void ALobbyGameMode::PostLogin(APlayerController *NewPlayer)
{
    Super::PostLogin(NewPlayer);
    NumberOfPlayers++;

    if (NumberOfPlayers >= 3)
    {
        UWorld *World = GetWorld();

        if (World == nullptr)
            return;

        bUseSeamlessTravel = true;
        World->ServerTravel("/Game/Maps/Game?listen");
    }
}
```

Transition Map은 아래와 같이 설정해두었다.

![25](/Assets/Images/Unreal/실습/PuzzlePlatforms/25.png)

### 세션에 일정 인원 모였을 시 시작 카운트 설정

사람이 꼭 세션에 가능한 전원이 모이지 않더라도 일정 수 이상 모이면 시작될 수 있게 구성하였다. 이를 위해 사용된 기능이 게임 플레이 타이머인데 이에 대해서는 아래에 자세히 정리해놓았다.

#### [GamePlay Timer 정리](/Unreal%20Engine/이론%20및%20정리/게임플레이%20타이머.md)

이를 활용해서 GameMode에서 게임 세션에 일정 사람이 모이면 SetTimer를 동작시켜 아래의 코드대로 10초 이후에 동작하게끔 타이머를 작동시켰다.

```C++

if (NumberOfPlayers >= 2)
    {

        GetWorldTimerManager().SetTimer(GameStartTimer, this, &ALobbyGameMode::StartGame, 10);
    }
```

### 게임이 시작 시 더이상 검색되지 않게하기

세션이 시작되고 나서는 더 이상 해당 세션은 검색되서는 안되기에 세션을 시작 처리를 해주었다.

세션 인터페이스를 통해 특정 세션을 시작하면 해당 세션은 더 이상 검색이 되지 않게된다.

세션 시작하는 함수

```C++
void UPuzzlePlatformsGameInstance::StartSession()
{
    if (SessionInterface.IsValid())
    {
        SessionInterface->StartSession(SESSION_NAME);
    }
}
```

### 서버측에 연결이 끝난다면?

플레이를 하던 중에 서버측에 연결이 끊길 때 클라이언트에 아무런 대비가 되지 않으면 클라이언트는 영문도 모른채 끊긴 화면을 보게 된다.

그렇기에 이를 해결하기 위해 서버측과의 연결이 끊기면 다시 메인메뉴 화면으로 돌아가게끔 구성했다.

모든 네트워크의 문제가 생겼을 때 처리를 할 수 있는 기능이 Unreal Engine에는 존재하는데 UnrealEngine에 내장된 기능인 OnNetworkFailure()가 그 기능이며 네트워크에 문제가 생겼을 시 동작될 델리게이트 함수를 이곳에 등록이 가능하다.

OnNetworkFailure에 함수 델리케이트를 등록했다.

```C++
if (GEngine != nullptr)
{
    GEngine->OnNetworkFailure().AddUObject(this, &UPuzzlePlatformsGameInstance::OnNetworkFailure);
}
```

이를 통해서 등록된 함수는 아래와 같이 메인 메뉴로 돌아가게끔 구성했다.

```C++
void UPuzzlePlatformsGameInstance::OnNetworkFailure(UWorld *World, UNetDriver *NetDriver, ENetworkFailure::Type FailureType, const FString &ErrorString = TEXT(""))
{
    LoadMainMenu();
}
```

이것으로 만약 서버측에서 문제가 생겨 게임을 지속할 수 없더라도 클라이언트들은 메인메뉴로 돌아가서 새로운 게임을 찾는 것이 가능해진다.
