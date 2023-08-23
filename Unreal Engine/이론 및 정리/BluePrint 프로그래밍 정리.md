# BluePrint 정리

이번 글에서는 언리얼 엔진에서의 블루프린트 기능을 학습하고 개인적으로 몰랐던 부분들을 정리해보겠습니다.

참고한 강의는 [언리얼 엔진 개발](https://www.udemy.com/course/unrealcourse-korean/)을 따라가며 배웠습니다.

## Data Pin & Execution Pin

![1](/Assets/Images/Unreal/이론/BluePrint정리/1.png)

그림에서 빨간색 상자 안에 있는 요소를 Data Pin이라 하며 노드에서의 데이터 입출력을 담당합니다.

하얀색 핀은 Execution Pin 으로 해당 노드가 언제 시작할 지를 결정합니다.

## 블루프린트 클래스 생성 시 네이밍 규칙

블루프린트 파생 클래스를 생성할 때는 해당 클래스가 블루프린트임을 쉽게 알게하기 위해 접두사로 'BP\_'를 붙이는 방식을 사용하는 습관을 들이는 것이 유용합니다.
