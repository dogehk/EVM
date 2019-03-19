해당 글은 Howard님이 2017/08/14 에 작성한 [Diving Into The Ethereum VM Part 2](https://medium.com/@hayeah/diving-into-the-ethereum-vm-part-2-storage-layout-bc5349cb11b7) 을 번역한 글입니다. 변역 과정에서 자연스러운 문장을 위해 다소 변경 된 부분이 존재할 수 있습니다.

* * *

## Part2 - How fixed-length data types are represented

```
contract C {
    uint256 a;
    function C() {
      a = 1;
    }
}
```

위의 contract는 part1에서 확인한 것처럼 "sstore" 명령어를 호출하게 된다.

```
// a = 1
sstore(0x0, 0x1)
```

    - EVM은 storage position "0x0"에 "0x1"을 저장한다.
    - 각 storage position은 32bytes (256 bits)를 저장할 수 있다.

우리는 Solidity가 structs, array와 같이 복잡한 데이터 타입을 가지는 데이터가 32bytes의 저장공간을 어떻게 사용하는지 알아볼 것이다. 또한 storage에 저장될 때 어떻게 최적화 되고 어떻게 저장이 실패하는지도 알아볼 것이다.
전통적인 프로그래밍 언어에서는 데이터 타입이 로우 레벨에서 어떻게 표현되는지 아는 것은 유용하지 않지만 Solidity 또는 다른 EVM 언어에서는 데이터 타입이 어떻게 표현되는지 중요하다. 그 이유는 storage 에 데이터를 저장하고 읽어오는 작업은 매우 비싼 gas가 들기때문이다. (아래는 storage에 접근하는 명령어에 따른 gas 소모량)

    - sstore은 20000 gas, 기본적인 산수 명령어보다 5000배 비싸다.
    - sload는 200 gas, 기본적인 산수 명령어보다 100배 비싸다.

앞으로 우리는 비용에 대해 성능의 속도가 아닌 지불되는 실제 돈(gas)를 가지고 이야기할 것이며, 결국 contract가 동작하는 데 많은 비용을 차지하는 부분은 "sstore", "sload"가 된다.


### Parsecs Upon Parsecs of Tape

Parsecs Upon Parsecs of Tape 에서는 EVM의 동작을 Universal Turing Machine에 비유하여 설명할 것이다.
<p align="center">
<img src="./images/TuringMachine.jpg" alt="Turing Machine. Source : http://raganwald.com/" class="center">(Fingure 1. Turing Machine. Source : http://raganwald.com/) </img>
</p>

Universal Turing Machine을 구축하기 위해서는 아래와 같은 두 가지 필수요소가 존재한다.

    - 점프 또는 재귀를 반복하는 방법(jump or recursion, loop)
    - 무한한 메모리

EVM 어셈블리 코드는 점프, 그리고 EVM storage가 제공하는 무한한 메모리를 가지며 이것으로 이더리움을 시뮬레이팅하기에 충분하다.

<p align="center">
<img src="./images/DIvingIntoTheMicroverseBattery.gif" alt="Diving Into The Microverse Battery
" class="center"><br>(Figure 2. Diving Into The Microverse Battery) </img>
</p>

contract를 위한 EVM storage는 Turing Machine의 무한한 종이 테이프와 같으며, 아래와 같이 테이프의 각 슬롯은 32 bytes를 가진다.

```
[32 bytes][32 bytes][32 bytes]...
```

이제 무한한 테이프(즉, 무한한 storage) 에서 데이터가 어떻게 존재하는지 확인할 것이다.
contract 당 테이프의 길이는 2<sup>256</sup> 또는 10<sup>77</sup> storage 슬롯을 가진다.

### The Blank Tape

storage는 최초에 모두 "0"으로 채워져있다. "0"으로 채워져 있는 경우 어떠한 비용(gas)도 발생하지 않는다. 왜 그런지 "0"(zero-value)가 동작하는 간단한 contract를 살펴보자.

```
pragma solidity ^0.4.11;
contract C {
    uint256 a;
    uint256 b;          uint256 c;
    uint256 d;
    uint256 e;
    uint256 f;
    function C() {
      f = 0xc0fefe;
    }
}
```

위의 contract는 아래와 같이 간단한 storage 레이아웃을 가진다.

    - a는 storage "0x0"
    - b는 storage "0x1"
    - c는 storage "0x2"
    - d는 storage "0x3"
    - e는 storage "0x4"
    - f는 storage "0x5"

위의 contract에서 살펴볼 핵심내용은 **"만약 우리가 f를 사용했을 때, 사용하지 않는 'a,b,c,d,e'를 위해 얼마의 비용을 지불해야 하는가?"** 이다.
자! 이제 위의 contract를 컴파일 해보자.

```
$ solc --bin --asm --optimize c-many-variables.sol
```

컴파일된 어셈블리 코드의 핵심은 아래와 같다.

```
// sstore(0x5, 0xc0fefe)
tag_2:
  0xc0fefe
  0x5
  sstore
```

위의 어셈블리 코드에서 보는 것과 같이 Store variables 선언 자체는 비용이 발생하지 않으며, 초기화 또한 필요하지 않다. 이처럼 Solidity는 변수를 위한 storage position을 준비하고, 그 storage position을 접근(사용)할 때만 비용(gas)을 지불하면 된다. 위의 예제의 경우 우리는 f 를 storage position "0x5"에 저장하기 위한 비용만 지불하면 된다.
만약, 직접 어셈블리 코드를 작성하는 경우 더 이상의 storage 확장 없이 어떤 storage postion에 저장할 지 선택할 수 있다.

### Reading Zero

storage 안의 position들은 언제든 쓰고(저장), 읽을 수(로드) 있으며, zero-value를 읽을 경우(초기화되지 않은 position)에는 "0x0"이 리턴된다.
아래는 초기화되지 않은 position을 읽는 contract 예제이다.

```
pragma solidity ^0.4.11;
contract C {
    uint256 a;
    function C() {
      a = a + 1;
    }
}
```

자! 위의 예제를 컴파일 해보자.

```
$ solc --bin --asm --optimize c-zero-value.sol
```

컴파일된 어셈블리 코드는 아래와 같다.

```
tag_2:
  // sload(0x0) returning 0x0
  0x0
  dup1
  sload
  // a + 1; where a == 0
  0x1
  add
  // sstore(0x0, a + 1)
  swap1
  sstore
```

위의 어셈블리 코드에서 확인할 수 있듯이 초기화되지 않은 storage position에서 데이터를 로드하는 코드("sload")가 유효하다. 그러나 우리는 Solidity 컴파일러보다 똑똑하기 때문에 "tag_2"가 생성자이고, a가 쓰여지지 않았다라는 것을 알고 "sload" 명령어를 "0x0"으로 대체하여 gas를 절약할 수 있다.


### Representing Struct

자~ 이제 본격적으로 struct type에 대해 살펴보자.
아래와 같이 6개의 필드를 가지는 복잡한 struct 타입이 존재하는 contract를 예로 들어보자.

```
pragma solidity ^0.4.11;
contract C {
    struct Tuple {
      uint256 a;
      uint256 b;
      uint256 c;
      uint256 d;
      uint256 e;
      uint256 f;
    }

    Tuple t;
    function C() {
      t.f = 0xC0FEFE;
    }
}
```

위의 예제와 같은 contract의 storage 레이아웃은 아래와 같다.

    - t.a는 storage "0x0"
    - t.b는 storage "0x1"
    - t.c는 storage "0x2"
    - t.d는 storage "0x3"
    - t.e는 storage "0x4"
    - t.f는 storage "0x5"

앞선 contract 예제와 같이 우리는 초기화를 위한 비용을 지불하지 않고 t.f에 직접 데이터를 쓸 수 있다.
이를 직접 컴파일하여 어셈블리 코드를 확인해보자.

```
$ solc --bin --asm --optimize c-struct-fields.sol
```

```
tag_2:
  0xc0fefe
  0x5
  sstore
```

### Fixed Length Array

이번엔 아래의 contract 예제를 통해 고정된 길이의 array 타입에 대해 살펴보자.

```
pragma solidity ^0.4.11;
contract C {
    uint256[6] numbers;
    function C() {
      numbers[5] = 0xC0FEFE;
    }
}
```

컴파일러는 정확하게 얼만큼의 uint256(32 bytes)가 있는지 알기 때문에, Store variables 및 구조체와 같은 배열의 요소를 하나씩 차례대로 배치할 수 있다.
위의 예제에서 우리는 stroage position "0x5"에 데이터를 저장하였고 이를 컴파일하여 어셈블리 코드로 확인해보자.

```
$ solc --bin --asm --optimize c-static-array.sol
```

```
tag_2:
  0xc0fefe
  0x0
  0x5
tag_4:
  add
  0x0
tag_5:
  pop
  sstore
```

어셈블리 코드가 조금 길어보이지만 자세히 살펴보면 앞선 예제와 동일하다. 위의 어셈블리 코드를 직접 최적화해보자.

```
tag_2:
  0xc0fefe
  // 0+5. Replace with 0x5
  0x0
  0x5
  add
  // Push then pop immediately. Useless, just remove.
  0x0
  pop
  sstore
```

태그와 의사 명령어를 제거하면, 결국에는 앞선 예제와 동일한 bytecode 시퀀스가 된다.

```
tag_2:
  0xc0fefe
  0x5
  sstore
```


### Array Bound Checking

고정된 길이를 가지는 array는 Store variables를 가지는 struct와 storage 레이아웃은 동일하지만 생성된 어셈블리 코드는 다르다.
그 이유는 Solidity가 array 접근을 위한 bound-checking 코드를 생성하기 때문이다.
자~ 그럼 array를 가지는 contract를 최적화 옵션을 끈 상태에서 컴파일해보자.

```
$ solc --bin --asm c-static-array.sol
```

컴파일된 어셈블리 코드는 아래와 같으며, 각 명령어를 수행한 이후의 machine 상태를 출력하여 주석으로 작성해 보았다.

```
tag_2:
  0xc0fefe
    [0xc0fefe]
  0x5
    [0x5 0xc0fefe]
  dup1
  /* array bound checking code */
  // 5 < 6
  0x6
    [0x6 0x5 0xc0fefe]
  dup2
    [0x5 0x6 0x5 0xc0fefe]
  lt
    [0x1 0x5 0xc0fefe]
  // bound_check_ok = 1 (TRUE)
  // if(bound_check_ok) { goto tag5 } else { invalid }
  tag_5
    [tag_5 0x1 0x5 0xc0fefe]
  jumpi
    // Test condition is true. Will goto tag_5.
    // And `jumpi` consumes two items from stack.
    [0x5 0xc0fefe]
  invalid
// Array access is valid. Do it.
// stack: [0x5 0xc0fefe]
tag_5:
  sstore
    []
    storage: { 0x5 => 0xc0fefe }
```

우리는 위의 어셈블리 코드에서 bound-checking 코드를 확인할 수 있다. 만약, 컴파일러가 일부를 최적화하지만 완벽하지는 않은 것을 볼 수 있다.
이제 우리는 array의 bound-checking가 어떻게 컴파일러의 최적화를 방해하여, 고정된 길이의 array이 store variables나 struct 보다 효율적이지 않은지를 확인해 볼 것이다.


### Packing Behaviour

storage 사용에는 매우 비싼 비용이 든다.(이것은 백번 천번 말해 왔다)
이 때문에 최적화의 한 가지 핵심은 32bytes 크기의 storage slot에 최대한 많은 데이터를 압축하여 넣는 것이다.

각 64bits를 가지는 4개의 store variables를 통해 256 bits (32bytes)를 추가한 constract를 살펴보자.

```
pragma solidity ^0.4.11;
contract C {
    uint64 a;
    uint64 b;
    uint64 c;
    uint64 d;
    function C() {
      a = 0xaaaa;
      b = 0xbbbb;
      c = 0xcccc;
      d = 0xdddd;
    }
}
```

우리는 컴파일러가 하나의 "sstore"을 이용해서 동일한 storage slot에 4개의 변수를 저장하기를 기대할 것이다.
실제 그러한지 확인해보자.

```
$ solc --bin --asm --optimize c-many-variables--packing.sol
```

```
tag_2:
    /* "c-many-variables--packing.sol":121:122  a */
  0x0
    /* "c-many-variables--packing.sol":121:131  a = 0xaaaa */
  dup1
  sload
    /* "c-many-variables--packing.sol":125:131  0xaaaa */
  0xaaaa
  not(0xffffffffffffffff)
    /* "c-many-variables--packing.sol":121:131  a = 0xaaaa */
  swap1
  swap2
  and
  or
  not(sub(exp(0x2, 0x80), exp(0x2, 0x40)))
    /* "c-many-variables--packing.sol":139:149  b = 0xbbbb */
  and
  0xbbbb0000000000000000
  or
  not(sub(exp(0x2, 0xc0), exp(0x2, 0x80)))
    /* "c-many-variables--packing.sol":157:167  c = 0xcccc */
  and
  0xcccc00000000000000000000000000000000
  or
  sub(exp(0x2, 0xc0), 0x1)
    /* "c-many-variables--packing.sol":175:185  d = 0xdddd */
  and
  0xdddd000000000000000000000000000000000000000000000000
  or
  swap1
  sstore
```

이해할 수 없는 bit-shuffling는 무시하고, **"sstore"이 하나만 사용되었으며 최적화에 성공했다.**


### Breaking The Optimizer

최적화기가 항상 잘 동작할 수 있다면, 그만하자. 우리가 할 수 있는 유일한 변화는 helper 함수를 이용하여 store variables를 설정하는 것뿐이다.

```
pragma solidity ^0.4.11;
contract C {
    uint64 a;
    uint64 b;
    uint64 c;
    uint64 d;
    function C() {
      setAB();
      setCD();
    }
    function setAB() internal {
      a = 0xaaaa;
      b = 0xbbbb;
    }
    function setCD() internal {
      c = 0xcccc;
      d = 0xdddd;
    }
}
```

```
$ solc --bin --asm --optimize c-many-variables--packing-helpers.sol
```

위의 contract를 컴파일한 어셈블리 코드는 너무 길기 때문에 우리는 상세한 부분은 무시하고 구조적인 부분에 초점을 맞출 것이다.

```
// Constructor function
tag_2:
  // ...
  // call setAB() by jumping to tag_5
  jump
tag_4:
  // ...
  // call setCD() by jumping to tag_7
  jump
// function setAB()
tag_5:
  // Bit-shuffle and set a, b
  // ...
  sstore
tag_9:
  jump  // return to caller of setAB()
// function setCD()
tag_7:
  // Bit-shuffle and set c, d
  // ...
  sstore
tag_10:
  jump  // return to caller of setCD()
```

위의 어셈블리 코드를 보면, 하나의 "sstore"가 아닌 두 개의 "sstore"가 사용되었다. 이는 Solidity 컴파일러는 tag 내부를 최적화할 수 있으나, tag들을 교차하여 최적화할 수 없기 때문이다.

함수 호출은 많은 비용이 들고, "sstore" 최적화에 실패하였기 때문에 함수를 호출하는 것은 비용이 많이 든다.

이 문제를 해결하기 위해, Solidity 컴파일러는 함수를 호출하지 않는 것과 동일한 코드를 얻을 수 있도록 함수들을 어떻게 inline화 할지 학습이 필요하다.

```
a = 0xaaaa;
b = 0xbbbb;
c = 0xcccc;
d = 0xdddd;
```

완성된 어셈블리 코드를 보면 setAB()와 setCD()함수에 "sstore" 명령어가 두 번 포함되어 코드 크기가 커지고, contract 동작에 추가 비용이 들게됩니다.
다음번에는 contract lifecycle에 대해 확인할 때 조금더 자세히 설명할 것이다.


### Why The Optimizer Breaks

최적화기는 tag들을 교차하여 최적화하지 않는다. "1+1" 수행하는 contract의 최적화 성공/실패에 대한 어셈블리 코드를 살펴 보자.

```
// Optimize OK!
tag_0:
  0x1
  0x1
  add
  ...
```

```
// Optimize Fail!
tag_0:
  0x1
  0x1
tag_1:
  add
  ...
```

위의 최적화 동작은 Solidity version 0.4.13 에서 추가됬다.


### Breaking The Optimizer, Again

다른 방법으로 최적화기가 실패한 것을 살펴보자. 고정된 길이의 array를 위해 packing이 동작하는가?
아래의 contract를 보자.

```
pragma solidity ^0.4.11;
contract C {
    uint64[4] numbers;
    function C() {
      numbers[0] = 0x0;
      numbers[1] = 0x1111;
      numbers[2] = 0x2222;
      numbers[3] = 0x3333;
    }
}
```

다시 말하지만 우리는 정확하게 하나의 "sstore" 명령어를 사용하기 위해, 4개의 64bits 변수를 32bytes storage slot에 packing하기를 원한다.
위의 contract를 컴파일한 어셈블리 코드는 너무 길기 때문에 어셈블리 코드 내의 "sstore", "sload" 명령어의 수만 확인하자.

```
$ solc --bin --asm --optimize c-static-array--packing.sol | grep -E '(sstore|sload)'
  sload
  sstore
  sload
  sstore
  sload
  sstore
  sload
  sstore
```

위의 "sstore", "sload"의 명령어 수(각각 4개씩 존재)를 보면 고정된 길이의 array가 struct 또는 store variables과 동일한 storage 레이아웃을 가지지만 최적화는 실패하였다.

어셈블리 코드를 간략하게 살펴보면 각 array access가 bound-checking 코드를가지고 있으며, 각각 다른 tag에 존재한다. 그러나 tag의 경계가 최적화를 깨버렸다.

한 가지 다행인 점은 아래와 같은 이유로 3개의 "sstore" 명령어는 첫 번째 "sstore" 명령어보다 비용이 적게 든다.

    - "sstore"는 새로운 storage position에 데이터를 쓸 때, 20000 gas
    - 이미 존재하는 storage position에 데이터를 쓸때, 5000 gas


### Conclusion

Solidity 컴파일러가 store variables의 크기를 파악할 수 있다면 storage에 차례로 저장한다. 그리고 가능한 데이터를 32bytes로 packing 한다.

요약하자면 packing의 동작은 아래와 같이 정리할 수 있다.

    - Store variables  : YES
    - Struct field : YES
    - Fixed-length arrays : No, 이론상으론 YES

storage를 사용하는 것은 비용이 많이 들기 때문에 데이터베이스 스키마를 구상할 때 store variables를 잘 생각해야한다.
contract를 작성할 때, 작은 단위로 코드를 작성하고 어셈블리 코드를 확인하여 컴파일러가 올바르게 최적화하였는지 확인하는 것도 유용하다.

우리는 Solidity 컴파일러가 앞으로 더 향상될 것이라 확신할 수 있다. 하지만 지금은 맹목적으로 최적화기를 신뢰할 수 없다.
