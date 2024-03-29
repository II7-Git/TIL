# GameInstance

게임 인스턴스란 게임이 시작될 때 엔진 시작 전에 생성되고 초기화되는 객체로 싱글톤 타입의 객체이다. 이는 게임이 시작될 때 유일하게 생성되서 관리된다는 뜻으로 게임에서 레벨을 이동해도 유일한 객체로 계속 존재하기에 게임에서 여러 레벨이나 모드에서 공유해야하는 영구적인 데이터 등을 관리하는 기능이나 콘솔을 통해서 게임의 전반 설정을 관리하는 방식으로 사용하기에 용이하다.

## 콘솔을 이용한 메소드 접근

GameInstance에서 만든 메소드는 콘솔을 통해서 접근이 가능하다.

아래와 같이 헤더 파일에서 UFUNCTION(Exec)을 통해 함수를 만들면 해당 함수의 콘솔을 통한 접근이 가능해진다.

```C++
UCLASS()
class PUZZLEPLATFORMS_API UPuzzlePlatformsGameInstance : public UGameInstance
{
	GENERATED_BODY()

public:
	UPuzzlePlatformsGameInstance(
		const FObjectInitializer &ObjectInitializer);

	virtual void Init() override;

	UFUNCTION(Exec)
	void Host();

	UFUNCTION(Exec)
	void Join(const FString &Address);
};
```

아래처럼 실제 Host 함수를 실행이 가능하다.
![7](/Assets/Images/Unreal/실습/PuzzlePlatforms/7.png)

이러한 기능을 통해서 게임에서 관리될 전반적인 설정 및 기능을 콘솔을 통해서 조작이 가능하게끔 해준다.
