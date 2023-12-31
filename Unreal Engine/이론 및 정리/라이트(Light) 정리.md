# Light 기능 정리

Unreal 에서 사용되는 라이트(light)의 기능들을 정리하려고 한다.

## Light의 5개의 종류

언리얼에는 다섯 개의 라이트가 존재한다.

![1](/Assets/Images/Unreal/이론/라이트%20정리/1.png)

### 디렉셔널 라이트

특정한 방향성을 가지고 전 맵에 동일한 방향으로 라이트가 비추게 된다. 그렇기에 어디에 배치되는지는 그다지 중요하지 않고 방향이 중요하다.<br>
그렇기에 주된 용도는 햇빛을 표현하는데 많이 사용된다.

![1](/Assets/Images/Unreal/이론/라이트%20정리/3.png)

### 포인트 라이트

전구처럼 특정 포인트에서 빛이 나오는 종류

가장 큰 특징은 광원이 하나라는 것이다.
![1](/Assets/Images/Unreal/이론/라이트%20정리/2.png)

### 스포트 라이트

이름답게 광원은 하나지만 그 광원에서 특정한 방향으로 방향성을 가진 빛이 나오게 된다.

특정 영역이나 객체를 주목할 때 사용

![1](/Assets/Images/Unreal/이론/라이트%20정리/4.png)

### 렉트 라이트

렉트 라이트는 빛이 나오는 영역이 좀 더 평면적으로 넓은 게 특징이다 해당 평면이 전체적으로 빛이 나와서 빛이 고르게 한쪽 방향으로 나아가는 것을 볼 수 있다.

![1](/Assets/Images/Unreal/이론/라이트%20정리/5.png)

### 스카이 라이트

말 그대로 하늘을 생성한 후 해당 객체에서 나오는 빛을 조절한다. 따라서 그냥 스카이 라이트만 배치해서는 빛이 나올 수 없다.

먼저 아래 사진처럼 하늘을 구성할 요소를 찾아서 레벨에 추가해준다.

![1](/Assets/Images/Unreal/이론/라이트%20정리/6.png)

BP_SkySphere 배치 전 후 차이는 아래 사진을 통해 확인 가능하다.
![1](/Assets/Images/Unreal/이론/라이트%20정리/7.png)
![1](/Assets/Images/Unreal/이론/라이트%20정리/8.png)

그 뒤 레벨에 스카이 라이트를 생성한 후 리캡처 버튼을 클릭해주면 레벨 구성 전체에 은은한 스카이 빛이 퍼지는 것을 확인할 수 있다.

![1](/Assets/Images/Unreal/이론/라이트%20정리/9.png)
![1](/Assets/Images/Unreal/이론/라이트%20정리/10.png)

### 스카이 라이트와 디렉셔널 라이트의 연계

두 라이트는 맵 전체에 빛춰지는 빛 속성이라는 것이 같은데 그렇기에 이를 연계해서 태양빛이 디렉셔널 라이트로 빛춰지는 것을 표현할 수 있다.

두 라이트를 각각 생성해준 뒤
BP_SkySphere에 Directional Light Actor에 디렉셔널 라이트를 연결해준다.

![1](/Assets/Images/Unreal/이론/라이트%20정리/11.png)

그 뒤 디렉셔널 라이트의 각도를 조절해주면서 Material을 다시 불러와주면서 확인하면 디렉셔널 라이트 각도에 따라 태양 위치가 이동하고 빛 방향과 빛 색깔이 변경되면서 자연스럽게 시간대에 따른 빛 연출을 할 수 있다.

낮 시간 색깔과 태양 위치

![1](/Assets/Images/Unreal/이론/라이트%20정리/12.png)

저녁 시간 색깔과 태양 위치

![1](/Assets/Images/Unreal/이론/라이트%20정리/13.png)

### 루멘(언리얼 5 기능) 사용 시 주의점

루멘 기능을 사용 시에 아래 사진에 있는 픽셀 뎁스 오프셋을 메테리얼에서 사용하면 안된다.
<br>그렇기에 아래 사진처럼 해당 기능의 연결을 끊어주어야한다.

![1](/Assets/Images/Unreal/이론/라이트%20정리/14.png)
