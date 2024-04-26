# PuzzlePlatforms(3)

## Session 설정

### Session생성 및 제거

Session을 생성 제거하는 방법으로는 SessionInterface를 통해서 세션이 존재하면 제거 아니면 생성하는 방식으로 온라인 플레이에서 사용할 세션을 생성 제거했다.

```C++
if (SessionInterface.IsValid())
    {
        auto ExistingSession = SessionInterface->GetNamedSession(SESSION_NAME);
        if (ExistingSession != nullptr)
        {
            SessionInterface->DestroySession(SESSION_NAME);
        }
        else
        {
            FOnlineSessionSettings SessionSettings;
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
