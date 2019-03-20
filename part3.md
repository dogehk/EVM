해당 글은 Howard님이 2017/08/24 에 작성한 [Diving Into The Ethereum VM Part 3](https://medium.com/@hayeah/diving-into-the-ethereum-vm-the-hidden-costs-of-arrays-28e119f04a9b) 을 번역한 글입니다. 변역 과정에서 자연스러운 문장을 위해 다소 변경된 부분이 존재할 수 있습니다.

* * *

## Part3 - How dynamic data types are represented.

Solidity는 다른 프로그래밍 언어에서 볼 수 있는 친숙한 데이터 구조를 제공한다. 숫자나 struct와 같은 데이터 타입이 아닌, 데이터가 추가됬을 때 동적으로 확장해주는 데이터 타입이 있다. 이러한 동적인 특성을 지니는 데이터 타입은 아래와 같이 3가지가 있다.

    - Mappings : mappings(bytes32 => uint256), mapping(address => string), etc.
    - Arrays : []uint256, []byte, etc.
    - Byte arrays. Only two kinds : string, bytes.

Part2에서 우리는 아래와 같이 고정된 크기를 가지는 타입이 어떻게 storage에서 나타나는지 확인했다.

    - Fundamental values : uint256, byte, etc.
    - Fixed sized arrays : [10]uint8, [32]byte, bytes32
    - 위의 타입들이 결합된 Structs

고정된 크기를 가지는 Storage variables는 storage에 차례로 저장되며, 가능한 32bytes의 크기로 packing 되어 있다.

이제 우리는 Solidity가 더 복잡한 데이터 구조들을 어떻게 지원하는지 알아볼 것이다. Solidity에서 Arrays와 mappings는 표면적으로는 친숙해보일 수 있지만 근본적으로는 다른 성능 특성이다.

우리는 셋 중에 가장 간단한 mapping을 먼저 살펴볼 것이다. array와 byte array들은 단지 멋지게 꾸민 mapping일 뿐이다.


### Mapping

```uint25 => uint256``` mapping에 하나의 값을 저장하는 contract를 통해 mapping의 동작을 살펴보자.

```
pragma solidity ^0.4.11;
contract C {
    mapping(uint256 => uint256) items;
    function C() {
      items[0xC0FEFE] = 0x42;
    }
}
```
```
solc --bin --asm --optimize c-mapping.sol
```

위의 contract를 컴파일하면 아래와 같은 어셈블리 코드가 생성된다.

```
tag_2:
  // Doesn't do anything. Should be optimized away.
  0xc0fefe
  0x0
  swap1
  dup2
  mstore
  0x20
  mstore

  // Storing 0x42 to the address 0x798...187c
  0x42
  0x79826054ee948a209ff4a6c9064d7398508d2c1909a392f899d301c6d232187c
  sstore
```

우리는 EVM이 key-value database로 저장하며, 각 key에는 최대 32bytes의 value를 저장할 수 있을 것이라 생각할 수 있다. 위의 예제를 살펴보면, key ```0xC0FEFE``` 를 직접 사용하지 않고, key는 ```0x798...187c```로 해쉬되고 값 ```0x42```가 여기에 저장된다. key 해쉬에는 ```keccak256 (SHA256)``` 함수가 사용된다.

위와 같은 간단한 예제에서 우리는 ```keccak256``` 명령어가 사용되는 것을 볼 수 없다. 그 이유는 최적화기가 미리 연산을 통해 결과를 bytecode 내에 포함시키기 때문이다. 우리는 여전히 유용하지 않은 ```mstore``` 명령어들에서 쓸모없은 계산을 볼 수 있다.


### Calculate The Address

python 코드를 통해 ```0xC0FEFE```를 ```0x798...187c```으로 해쉬해보자. ```keccak_256```을 사용하기 위해서 "python 3.6", "pysha3"가 필요하다.

먼저, 두 개의 helper 함수를 정의하자.

```
import binascii
import sha3

# Convert a number to 32 bytes array.
def bytes32(i):
    return binascii.unhexlify('%064x' % i)

# Calculate the keccak256 hash of a 32 bytes array.
def keccak256(x):
    return sha3.keccak_256(x).hexdigest()
```

정의한 helper 함수(```bytes32```)를 통해 숫자들을 32bytes로 변환하면 아래와 같다.

```
>>> bytes32(1)
b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01'

>>> bytes32(0xC0FEFE)
b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xc0\xfe\xfe'
```

```"+"``` 연산자를 사용하여 두 byte array를 아래와 같이 연결할 수 있다.

```
>>> bytes32(1) + bytes32(2)
b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02'
```

```bytes(1)```를 정의한 helper 함수(```keccak256```)를 통해 해쉬하면 아래와 같다.

```
>>> keccak256(bytes(1))
'bc36789e7a1e281436464229828f817d6612f7b477d66591ff96a9e064bcc98a'
```

이제 우리는 ```0x798...187c``` 를 계산할 수 있다.

store variable ```items```의 store position은 ```"0x0"```이다. (```items```이 첫 번째 store variable 일 때)
주소를 얻기 위해서 key ```0xc0fefe```와 ```items```의 position ```"0x0"을 아래와 같이 조합하자.

```
# key = 0xC0FEFE, position = 0
>>> keccak256(bytes32(0xC0FEFE) + bytes32(0))
'79826054ee948a209ff4a6c9064d7398508d2c1909a392f899d301c6d232187c'
```

이렇게 hash된 key를 구할 수 있으며 이를 공식화하면 아래와 같이 키의 저장 주소를 구하기 위한 계산 공식이 만들어진다.

```
keccak256(bytes32(key) + bytes32(position))
```


### Two Mappings

