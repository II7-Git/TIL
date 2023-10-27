# ToonTanks(5)

## Particle 적용

### UParticleSystem

당연히 C++ 코드에서도 파티클 시스템을 사용할 수 있는 클래스가 존재하고 이를 UParticleSystem이라고 한다.

[UParticleSystem Docs](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Particles/UParticleSystem/)

위 링크의 Docs를 참조해보면 다양한 기능들을 지원해주는 것을 알 수 있다.

### 파티클 소환 : SpawnEmitterAtLocation()

파티클이 준비됐다면 UGameplayStatics에 있는 SpawnEmitterAtLocation()함수를 이용하여 파티클을 소환하는 것이 가능하다. 이를 필요한 상황에 맞게 적용시켜준다. 이 때 SpawnEmitterAtLocation()를 사용하면 파티클이 끝나고 사망하는 처리까지 알아서 해주기에 평범한 액터 스폰 함수를 사용하지말고 저 함수를 사용하는 것이 맞다.

만약 파티클이 있다면 파티클을 현재 액터에 위치와 회전과 같은 방향으로 소환

```C++
if (HitParticles)
{
	UGameplayStatics::SpawnEmitterAtLocation(this, HitParticles, GetActorLocation(), GetActorRotation());
}
```

### UParticleSystemComponent

[UParticleSystemComponent Docs](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Particles/UParticleSystemComponent/)

앞선 기능은 파티클이 생성되고 사라지는 경우에 사용하면 좋다. 반면에 파티클이 계속해서 유지되어야할 때는 어떠한 방식을 취하면 좋을까?

답은 파티클을 컴포넌트화 시켜서 연결시키는 방식이다. 이 때 사용하는 것이 UParticleSystemComponent 이다.

아래처럼 생성자 부분에서 컴포넌트로 생성해주고 원하는 Component 밑에 붙힌다.

```C++
AProjectile::AProjectile()
{
	TrailParticles = CreateDefaultSubobject<UParticleSystemComponent>(TEXT("Smoke Trail"));
	TrailParticles->SetupAttachment(RootComponent);
}
```

### 구현 모습

실제 구현해서 각 이펙트들이 적용된 모습이다.

충돌 위치 파티클,발사체 궤적 파티클
![21](/Assets/Images/Unreal/실습/ToonTanks/21.png)

파괴 파티클
![22](/Assets/Images/Unreal/실습/ToonTanks/22.png)

## Sound 적용

### [USoundBase](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Sound/USoundBase/)

소리에서 사용되는 클래스이다.

### [PlaySoundAtLocation](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Kismet/UGameplayStatics/PlaySoundAtLocation/2/)

사운드를 원하는 위치에서 재생시킬 땐 PlaySoundAtLocation()을 사용한다.

사용 방식은 아래와 같이 인풋값으로는 (WorldObject,USoundBase,Location)이 들어가며 실행되면 USoundBase의 소리가 해당 위치에서 들리게 된다.

```C++
if (HitSound)
		{
			UGameplayStatics::PlaySoundAtLocation(this, HitSound, GetActorLocation());
		}
```

## 카메라 셰이크

플레이에 박진감있는 연출을 위해서 카메라에 특수 효과를 줄 수 있는데 이번에는 그 중에 미사일이 대상을 맞히면 카메라가 흔들리는 기능을 넣어보려 한다.

### [ULegacyCameraShake](https://docs.unrealengine.com/5.3/en-US/API/Plugins/GameplayCameras/ULegacyCameraShake/)

언리얼에서는 카메라의 흔들림을 지원하는데 이번에는 그 중에서 ULegacyCameraShake를 사용해보았다.

![23](/Assets/Images/Unreal/실습/ToonTanks/23.png)

해당 클래스를 생성하면 다양한 변수를 통한 흔들림 옵션을 확인할 수 있다
![24](/Assets/Images/Unreal/실습/ToonTanks/24.png)

### [UCameraShakeBase](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Camera/UCameraShakeBase/)

위에서 만든 클래스를 사용할 때 UCameraShakeBase를 TSubClassOf<>를 사용해서 코드에서 클래스를 등록하게 된다.

### ClientStartCameraShake()

실제 코드에서 실행은 Controller에 있는 ClientStartCameraShake()로 실행한다.

### 실행 코드

```C++
//선언 부분
UPROPERTY(EditAnywhere, Category = "Combat")
TSubclassOf<class UCameraShakeBase> HitCameraShakeClass;

//실행 부분

if (HitCameraShakeClass)
{
	GetWorld()->GetFirstPlayerController()->ClientStartCameraShake(HitCameraShakeClass);
}
```
