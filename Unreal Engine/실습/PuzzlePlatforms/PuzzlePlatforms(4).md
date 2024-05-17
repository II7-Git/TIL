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
