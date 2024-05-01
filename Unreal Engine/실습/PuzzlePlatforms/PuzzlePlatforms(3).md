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
