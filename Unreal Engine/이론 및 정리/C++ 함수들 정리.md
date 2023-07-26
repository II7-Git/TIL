# Unreal Engine Method

언리얼 엔진에서 지원하는 다양한 함수들 중에 자주 쓰이게 되는 것들이 있다는 것을 알게 됐고 그에따라 이러한 메소드들은 정리가 필요할 것 같다고 생각했습니다. 그래서 어떤 기능을 가졌는지 하나씩 정리해보려 합니다.

---

## Method

- virtual void PostInitProperties() override

  > 언리얼엔진에서 오브젝트의 변수가 초기화될 때 호출되는 함수입니다.
  > <br>이 함수를 통해서 변수를 초기화 할 때 추가적인 작업을 진행할 수 있습니다.
  > <br>사용하려면 항상 오버라이드해줘야 합니다.

- virtual void PostEditChangeProperty(FPropertyChangedEvent& PropertyChangeEvent) override
  > 변수의 값이 수정될 때 호출되는 함수입니다.
  > <br>변수가 변경될 때마다 사용할 로직들을 이곳에서 사용하면 매우 효과적입니다.
  > <br>사용하려면 항상 오버라이드해줘야 합니다.
