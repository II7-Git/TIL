# Static Load
에셋을 빌드, 실행 시간에 고정적으로 메모리에 로드하는 방식 
    > 특징
    > 직접 참조(Direct Reference) :  에셋이 다른 객체(블루프린트, 레벨)에서 직접적으로 참조
    > 예 : UPROPERTY 를 통해 코드에서 참조되거 블루프린트에서 하드 레퍼런스된 에셋
    > 항상 로드 : 사용 여부와 상관없이 실행 시 메모리에 로드되기에 불필요한 사용 유발 가능
    > 빠른 접근 속도 : 이미 로드되어있기에 에셋 접근이 매우 빠름
    > 의존성 증가 : 직접 참조는 에셋 간 의존성을 높여서 특정 에셋의 제거,변경하기 어려움
    > 빠른 로드와 사용이 장점이기에 중요한 에셋( HUD, 플레이어 캐릭터, UI) 등에서 사용

# Soft Load
에셋을 필요할 때만 동적으로 메모리에 로드하는 방식
    > 특징
    > 간접 참조(Indirect Reference): <code>TSoftObjectPtr</code> or <code>TSoftClassPtr</code> 을 통해서 에셋을 참조
    > 블루프린트에서 소프트 레퍼런스를 통해 지정 가능
    > 지연 로드 (Lazy Loading) : 에셋이 필요할 때만 메모리에 로드
    > 메모리 사용량이 감소하여 효율적 관리 가능
    > 로드 지연 : 에셋이 로드되는 시간이 걸릴 수 있어, 로드 완료를 기다려야 사용 가능
    > 의존성 낮음 : 에셋 간의 결합도가 낮아지고 관리가 쉬워짐
    > 장점 : 메모리 절약, 에셋 관리 및 업데이트가 용이, 대규모 게임에서 효율적
    > 단점 : 로드 시 딜레이가 발생할 수 있음, 로드 실패 시 에러 처리 필요
```C++
TSoftObjectPtr<UStaticMesh> SoftMesh = TSoftObjectPtr<UStaticMesh>(FSoftObjectPath(TEXT("/Game/Meshes/MyMesh.MyMesh")));
if (SoftMesh.IsValid())
{
    UStaticMesh* Mesh = SoftMesh.Get(); // 필요할 때 로드
}
```
