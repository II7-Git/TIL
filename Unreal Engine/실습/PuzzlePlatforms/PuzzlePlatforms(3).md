# PuzzlePlatforms(3)

## Session 설정

### Session생성 및 제거

Session을 생성 제거하는 방법으로는 FOnlineSessionSettings 구조체를 통해서 온라인 세팅에서 설정할 옵션들을 세팅해주고 이를 바탕으로 세션을 생성하는 작업으로 진행했다.

```C++
void UPuzzlePlatformsGameInstance::CreateSession()
{
    if (SessionInterface.IsValid())
    {
        FOnlineSessionSettings SessionSettings;
        // 로컬 네트워크를 통한 검색 허용, 같은 컴퓨터에서 매칭 가능하게 한다.
        SessionSettings.bIsLANMatch = true;
        // 플레이어 수 제한 (public, private 모두 있다)
        SessionSettings.NumPublicConnections = 2;
        // 온라인에서 세션을 볼 수 있게하는 옵션
        SessionSettings.bShouldAdvertise = true;

        SessionInterface->CreateSession(0, SESSION_NAME, SessionSettings);
    }
}
```

세션을 생성 제거할 때 클래스에서 동작할 추가 메소드들을 각 상황에 맞춰 델리게이트 함수로 등록해놓아서 세션이 생성 및 제거 될 시에 추가 작업을 진행할 수 있게 구성해놓았다.

```C++
 SessionInterface = Subsystem->GetSessionInterface();
    if (SessionInterface.IsValid())
    {
        SessionInterface->OnCreateSessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnCreateSessionComplete);
        SessionInterface->OnDestroySessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnDestroySessionComplete);
    }
```

### Session 현재 리스트를 찾아서 출력

Session을 생성 제거를 끝냈으면 이를 바탕으로 Session이 생성된 정보들을 받아와서 원하는 방에 접속할 수 있도록 구성해야한다. 이를 위해 스크롤 박스에 현재 생성된 세션의 ID를 받아와서 출력하는 작업을 진행했다.

먼저 GameInstance에서는 세션 찾는 함수를 비동기함수로 등록해준다.

```C++
 SessionInterface = Subsystem->GetSessionInterface();
    if (SessionInterface.IsValid())
    {
        SessionInterface->OnCreateSessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnCreateSessionComplete);
        SessionInterface->OnDestroySessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnDestroySessionComplete);
        SessionInterface->OnFindSessionsCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnFindSessionsComplete);
    }
```

이 뒤에 유저의 사용 흐름대로 따라가보면

MainMenu에서 Join버튼을 누르면 MenuInterface를 통해 게임 인스턴스의 RefreshServerList()가 호출된다.

MainMenu.cpp

```C++
void UMainMenu::OpenJoinMenu()
{
    if (JoinMenu == nullptr)
        return;

    MenuSwitcher->SetActiveWidget(JoinMenu);

    if (MenuInterface != nullptr)
    {
        MenuInterface->RefreshServerList();
    }
}
```

게임 인스턴스에서는 공유 포인터를 만들어서 찾은 세션 목록을 SessionSearch가 가르키게 한다

PuzzlePlatformsGameInstance.cpp

```C++
void UPuzzlePlatformsGameInstance::RefreshServerList()
{
    SessionSearch = MakeShareable(new FOnlineSessionSearch());
    if (SessionSearch.IsValid())
    {
        // 로컬 내트워크 매칭 허용
        SessionSearch->bIsLanQuery = true;

        UE_LOG(LogTemp, Warning, TEXT("Starting Find Session"));
        SessionInterface->FindSessions(0, SessionSearch.ToSharedRef());
    }
}
```

파인드 세션이 호출됐기에 비동기 함수로 등록해놓은 OnFindSessionsComplete가 호출된다.<br>
이때 ServerNames 배열에 생성된 세션의 ID를 추가하고 이를 Menu에 SetServerList에 인자로 전달한다.

```C++
void UPuzzlePlatformsGameInstance::OnFindSessionsComplete(bool Success)
{

    if (Success && SessionSearch.IsValid() && Menu != nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("Finished Find Session"));

        TArray<FString> ServerNames;

        for (const FOnlineSessionSearchResult &SearchResult : SessionSearch->SearchResults)
        {
            UE_LOG(LogTemp, Warning, TEXT("Found session names: %s"), *SearchResult.GetSessionIdStr());
            ServerNames.Add(SearchResult.GetSessionIdStr());
        }

        Menu->SetServerList(ServerNames);
    }
}
```

MainMenu.cpp

먼저 현재 ServerList의 자식을 초기화하여 기존 세션 목록을 지우고 새롭게 갱신한 세션 ID들을 하위 자식으로 추가해준다.

```C++
void UMainMenu::SetServerList(TArray<FString> ServerNames)
{

    UWorld *World = this->GetWorld();
    if (World == nullptr || ServerRowClass == nullptr)
        return;

    // 새로 서버 리스트를 설치하기 위해 기존 리스트 제거
    ServerList->ClearChildren();

    for (const FString &ServerName : ServerNames)
    {
        UServerRow *Row = CreateWidget<UServerRow>(World, ServerRowClass);

        if (Row == nullptr)
            return;

        Row->ServerName->SetText(FText::FromString(ServerName));

        ServerList->AddChild(Row);
    }
}

```

위 작업을 끝마치고 실행해보면 실제 아래와 같이 세션 ID를 가져와서 화면에 출력하는 모습을 확인할 수 있다.

두 개의 Host를 생성하고 Join을 누르면 세션 정보 확인했다.
![16](/Assets/Images/Unreal/실습/PuzzlePlatforms/16.png)

Host를 종료하면 세션 사라진 것을 확인했다.
![17](/Assets/Images/Unreal/실습/PuzzlePlatforms/17.png)

로그를 통해서 마찬가지로 확인 가능했다.
![18](/Assets/Images/Unreal/실습/PuzzlePlatforms/18.png)

### Session Join 기능을 리스트에서 선택한 정보를 통해서 구현

기존에 구현했던 Session Join 방식을 클릭한 세션 정보를 통해서 접속을 시도하는 구조로 만들어 방을 통한 접속 기능을 구현하고자 했다. 세션의 접속하고서 진행돼야 하는 후속 처리는 OnlineSession의 기능인 OnJoinSessionComplete에 비동기 함수로 등록해서 진행해줬다.

MainMenu에서 JoinServer 버튼을 누르면 GameInstance에 Join기능을 실행한다. 이때 SelectedIndex에는 내가 클릭한 세션의 Index 값이 들어가서 전해진다.

```C++
void UMainMenu::JoinServer()
{
    if (SelectedIndex.IsSet() && MenuInterface != nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("Selected index %d"), SelectedIndex.GetValue());
        MenuInterface->Join(SelectedIndex.GetValue());
    }
    else
    {
        UE_LOG(LogTemp, Warning, TEXT("Selected index not set."));
    }
}

```

GameInstance에 Join 구현을 보면 받은 Index를 통해서 생성된 세션 중 Index위치의 세션에 대해 접속을 시도한다.

```C++
void UPuzzlePlatformsGameInstance::Join(uint32 Index)
{
    if (!SessionInterface.IsValid())
        return;

    if (!SessionSearch.IsValid())
        return;

    if (Menu != nullptr)
    {
        // Menu->SetServerList({});
        Menu->Teardown();
    }

    SessionInterface->JoinSession(0, SESSION_NAME, SessionSearch->SearchResults[Index]);
}
```

세션 접속이 성공했다면 아래의 코드를 통해 세션 접속이 성공했을시 발동되는 비동기 함수를 동작시킨다.

```C++
SessionInterface->OnJoinSessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnJoinSessionComplete);
```

비동기 함수에서는 세션 접속 성공 시 실행돼야할 후속 처리를 진행한다. 여기서는 세션에 대한 정보를 가져와서 이를 통해서 세션에 대한 Address를 얻어와서 ClientTravel을 통해서 실제 레벨 이동을 하여서 Join한 플레이어가 로비에서 벗어나 플레이어들이 같은 레벨에서 만날 수 있게 동작시킨다.

```C++
void UPuzzlePlatformsGameInstance::OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result)
{
    if (!SessionInterface.IsValid())
        return;

    FString Address;
    if (!SessionInterface->GetResolvedConnectString(SessionName, Address))
    {
        UE_LOG(LogTemp, Warning, TEXT("Could not get connect String"));
        return;
    }

    UEngine *Engine = GetEngine();
    if (Engine == nullptr)
        return;

    Engine->AddOnScreenDebugMessage(0, 20, FColor::Green, FString::Printf(TEXT("Joining %s"), *Address));

    APlayerController *PlayerController = GetFirstLocalPlayerController();

    if (PlayerController == nullptr)
        return;

    PlayerController->ClientTravel(Address, ETravelType::TRAVEL_Absolute);
}

```

이를 통해서 최종 구현된 모습

왼쪽에서 방 정보 확인
![19](/Assets/Images/Unreal/실습/PuzzlePlatforms/19.png)

이를 클릭하면 실제 접속 확인
![20](/Assets/Images/Unreal/실습/PuzzlePlatforms/20.png)

### SteamOSS 활성화 및 연결

OnlineSubSystem(OSS)로 스팀을 이용하여 스팀 서버를 통해서 연결하는 방법을 구현하려한다.

Build.cs 에는 "OnlineSubsystemSteam" 모듈을 추가해준다.

```C++
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "HeadMountedDisplay", "EnhancedInput","UMG" ,"OnlineSubsystem","OnlineSubsystemSteam"});
```

프로젝트 폴더로 가서 DefaultEngine.ini에 아래 설정을 하단에 추가해준다.

DefaultEngine.ini

```

	[/Script/Engine.GameEngine]
	+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="OnlineSubsystemSteam.SteamNetDriver",DriverClassNameFallback="OnlineSubsystemUtils.IpNetDriver")

	[OnlineSubsystem]
	DefaultPlatformService=Steam

	[OnlineSubsystemSteam]
	bEnabled=true
	SteamDevAppId=480

	; If using Sessions
	; bInitServerOnClient=true

	[/Script/OnlineSubsystemSteam.SteamNetDriver]
	NetConnectionClassName="OnlineSubsystemSteam.SteamNetConnection"
```

이를 통해 사용할 준비는 끝났고 실제 세션 연결 부분에서 약간의 설정을 해준다.

세션을 만들어주는 CreateSession 부분에서는 스팀의 로비 기능을 사용하기 위해 bUsesPresence 옵션을 켜준다.

```C++
SessionSettings.bUsesPresence = true;
```

세션을 찾는 부분에서는 쿼리 세팅을 통해서 PRESENCE를 통해서 스팀 로비를 통해 찾는 옵션을 세팅해준다.

```C++

SessionSearch->MaxSearchResults = 100;
SessionSearch->QuerySettings.Set(SEARCH_PRESENCE, true, EOnlineComparisonOp::Equals);
```

위 과정을 통해서 OSS가 스팀을 통해서 세션을 찾을 수 있게 된다.

### 서버 Join UI 다듬기

서버 리스트에서 어떠한 세션을 선택했는지 애매한 부분이 있기에 이를 클릭한 서버의 글씨색을 바꾸어서 유저가 구분하기 편하게 설정하였다.

![21](/Assets/Images/Unreal/실습/PuzzlePlatforms/21.png)

기존에는 세션의 이름만 나타나고 다른 정보는 확인할 수 없었다. 이를 바꾸기 위해 SessionResult에서 가져올 수 있는 정보들을 바탕으로 구조체를 만들어서 추가적인 세션에 정보를 띄우기로 하였다.

세션창에 띄울 정보들을 구조체로 만들었다.

```C++
USTRUCT()
struct FServerData
{
	GENERATED_BODY()

	FString Name;
	uint16 CurrentPlayers;
	uint16 MaxPlayers;
	FString HostUsername;
};
```

게임 인스턴스에서 세션에서 얻은 FOnlineSessionSearchResult의 정보들로 필요한 구조체 정보들을 채워서 서버 리스트에 추가하였다.

```C++
for (const FOnlineSessionSearchResult &SearchResult : SessionSearch->SearchResults)
        {
            UE_LOG(LogTemp, Warning, TEXT("Found session names: %s"), *SearchResult.GetSessionIdStr());
            FServerData Data;
            Data.Name = SearchResult.GetSessionIdStr();
            Data.MaxPlayers = SearchResult.Session.SessionSettings.NumPublicConnections;
            Data.CurrentPlayers = Data.MaxPlayers - SearchResult.Session.NumOpenPublicConnections;
            Data.HostUsername = SearchResult.Session.OwningUserName;
            ServerNames.Add(Data);
        }
```

이를 통해서 각종 정보를 세션 리스트에서 확인이 가능해졌다.
![22](/Assets/Images/Unreal/실습/PuzzlePlatforms/22.png)

### 방 제목 설정

FOnlineSessionSettings에 Set 함수를 통해서 특정 원하는 변수를 전달하는 것이 가능하다 Key값을 맞춰주고 원하는 정보를 넘겨서 이를 같은 세션 정보에서 Get()을 통해 가져오면 원하는 정보를 세션을 통해서 주고 받는 것이 가능하다. 이를 이용해서 방 제목 설정을 진행해보았다.

가장 중요한 부분만 보면 SessionSettings.Set(FName Key, const FString &Value, EOnlineDataAdvertisementType::Type InType) 부분으로 임의의 설정한 Key값과 전해줄 Value를 설정하여 세션에 저장한다.

```C++
SessionSettings.Set(SERVER_NAME_SETTING_KEY, DesiredServerName, EOnlineDataAdvertisementType::ViaOnlineServiceAndPing);
```

그러고 난 뒤 세션을 얻은 부분에서는 Get(FName Key, FString &Value)을 통해서 Key를 통해서 값을 찾고 Value에 참조 연산으로 저장한다.

```C++
FString ServerName;
            if (SearchResult.Session.SessionSettings.Get(SERVER_NAME_SETTING_KEY, ServerName))
            {
                // UE_LOG(LogTemp, Warning, TEXT("Data Found in Settings: %s"), *ServerName);
                Data.Name = ServerName;
            }
            else
            {
                // UE_LOG(LogTemp, Warning, TEXT("Didnt get expected Data"));
                Data.Name = "Could not find name";
            }
```

이를 통해서 방 제목을 Host할 때 설정하고 Find할 때 해당 방 제목을 보는 것이 가능하다.

방 제목을 설정하는 장면

![23](/Assets/Images/Unreal/실습/PuzzlePlatforms/23.png)

해당 방 제목을 확인 가능하다.

![24](/Assets/Images/Unreal/실습/PuzzlePlatforms/24.png)
