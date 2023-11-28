# SimpleShooter 6

## 사운드

### 사운드 큐

소리를 다양하게 해줄 수 있는 언리얼에서 제공되는 기능으로 애니메이션 설정처럼 블루프린트를 통해서 설정이 가능하다.

아래처럼 랜덤한 소리를 재생하고 모듈레이터로 소리의 크기나 피치를 변환시켜줌으로써 다채로운 사운드를 재생 가능하게 했다.

![32](/Assets/Images/Unreal/실습/SimpleShooter/32.png)

### 사운드 공간화

사운드가 거리나 방향에 따라 소리가 다르게 들리는 것을 구현하는 기능이다.

사운드 큐에서 사운드 어테뉴에이션을 설정해 줄 수 있는데 이를 통해서 방향감을 구현한다. 이를 따로 분리하여 구현하여 여러 사운드 큐에서 같은 설정을 사용하는 것도 가능하다.
![34](/Assets/Images/Unreal/실습/SimpleShooter/34.png)
![33](/Assets/Images/Unreal/실습/SimpleShooter/33.png)

### 소리 발생

총기 발사 할 때랑 맞는 곳 위치에 Effect를 설정해놓았는데 그 위치에 사운드도 넣어준다.

```C++
// 총구에서 발사될 때 소리 부착
UGameplayStatics::SpawnEmitterAttached(MuzzleFlash, Mesh, TEXT("MuzzleFlashSocket"));
UGameplayStatics::SpawnSoundAttached(MuzzleSound, Mesh, TEXT("MuzzleFlashSocket"));

// 발사된 탄 맞는 위치에 소리 생성
UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ImpactEffect, Hit.Location, ShotDirection.Rotation());
UGameplayStatics::SpawnSoundAtLocation(GetWorld(), ImpactSound, Hit.Location, ShotDirection.Rotation());
```
