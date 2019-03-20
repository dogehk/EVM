해당 글은 Howard님이 2017/08/24 에 작성한 [Diving Into The Ethereum VM Part 3](https://medium.com/@hayeah/diving-into-the-ethereum-vm-the-hidden-costs-of-arrays-28e119f04a9b) 을 번역한 글입니다. 변역 과정에서 자연스러운 문장을 위해 다소 변경 부분이 존재할 수 있습니다.

* * *

## Part3 - How dynamic data types are represented.

Solidity는 다른 프로그래밍 언어에서 볼 수 있는 친숙한 데이터 구조를 제공한다. 숫자나 struct와 같은 데이터 타입이 아닌, 데이터가 추가됬을 때 동적으로 확장해주는 데이터 타입이 있다. 이러한 동적인 특성을 지니는 데이터 타입은 아래와 같이 3가지가 있다.

    - Mappings : ```mappings(bytes32 => uint256), mapping(address => string)```, etc.
    - Arrays : ```[]uint256, []byte```, etc.
    - Byte arrays. Only two kinds : ```string, bytes```.

Part2에서 우리는 아래와 같이 고정된 크기를 가지는 타입이 어떻게 storage에서 나타나는지 확인했다.

    - Fundamental values : ```uint256, byte```, etc.
    - Fixed sized arrays : ```[10]uint8, [32]byte, bytes32```
    - 위의 타입들이 결합된 Structs

고정된 크기를 가지는 Storage variables는 storage에 차례로 저장되며, 가능한 32bytes의 크기로 packing 되어 있다.

이제 우리는 Solidity가 더 복잡한 데이터 구조들을 어떻게 지원하는지 알아볼 것이다. Solidity에서 Arrays와 mappings는 표면적으로는 친숙해보일 수 있지만 근본적으로는 다른 성능 특성이다.

우리는 셋 중에 가장 간단한 mapping을 먼저 살펴볼 것이다. array와 byte array들은 단지 멋지게 꾸민 mapping일 뿐이다.


### Mapping

