# Seamless Travel

언리얼에서는 맵을 Travel하는 방식이 Seamless와 non-seamless로 나뉘는데 non-seamless방식은 맵을 이동할 때 프리징이 생겨서 맵간에 이동 상황에서 적절하지 않은 방식이다. 그래서 non-seamless는 아래 정도의 경우에서만 쓰이고 대부분은 seamless 방식으로 구현하는 것이 좋다.

#### non-seamless의 예

- 맵을 처음 로드할 때
- 서버에 클라이언트로 처음 접속할 때
- 멀티플레이어 게임을 끝내고 새 게임을 시작할 때

## Seamleass Travel 이란?

### [Seamless Travel Doc](https://dev.epicgames.com/documentation/en-us/unreal-engine/travelling-in-multiplayer-in-unreal-engine)

심리스 이동 방식은 두 개의 맵 사이를 매끄럽게 이동하는 방법으로 기존에 방식은 한 맵에서 다른 맵으로 이동할 때 두 맵의 메모리가 크다면 반드시 렉이 생길 수 밖에 없다.

반면 심리스는 중간에 오버헤드가 안생길 정도로 작은 맵을 하나 두어서 해당 맵을 거쳐서 이동하는 방식이다.

'Map A' 에서 'Map B'로 이동한다면 아래와 같은 구조로 이동하게 되는 것이 Seamless 방식이다.

```
Map A -> Transition Map(오버헤드가 없을만큼 작은 맵) -> Map B
```

이를 통해서 Transition Map은 로딩 속도가 빠르기에 바로 이동되고 Map A는 지워졌기에 자원의 확보가 된 상태에서 Map B가 로딩이 완료되면 Map B로 매끄럽게 이동할 수 있게된다.

## Seamless 구현 방법

Seamless의 옵션은 해당 레벨에 GameMode에서 On Off를 설정할 수 있다.

GameMode에서 아래의 변수를 통해 조절한다.

```C++
bUseSeamlessTravel = true;
```

Transition Map은 프로젝트 세팅에서 설정이 가능하다.

![25](/Assets/Images/Unreal/실습/PuzzlePlatforms/25.png)
