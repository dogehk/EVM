## Part1 - Introduction to the EVM Assembly Code
```
해당 글은 Howard님이 2017/08/06 에 작성한 [Diving Into The Ethereum Virtual Machine](https://blog.qtum.org/diving-into-the-ethereum-vm-6e8d5d2f3c30) 을 번역한 글입니다. 변역 과정에서 자연스러운 문장을 위해 다소 변경한 부분이 존재할 수 있습니다. 
```

Solidity는 고급 프로그래밍 언어로 많은 추상화를 제공한다. 그러나 이러한 추상화는 Solidity 관련 문서를 읽어봐도 프로그램이 동작하는 동안에 무슨일이 일어나는지 이해하기 어렵다. 예를 들면 아래와 같은 것들 말이다.

```
1. String, bytes32, byte[ ], bytes의 차이점은 무엇인가?
- 언제 어떤 것을 선택해서 사용해야하는 가?
- string에서 bytes로 변환할 때 어떤 일이 발생하는 가? byte[ ]로 변환할 수 있는가?
- 얼마나 많은 비용이 드는가?

2. mapping 들은 EVM에 의해 어떻게 저장되는가?
- 왜 mapping을 지울수 없는가?
- mapping의 mapping을 가지는 것은 어떻게 동작하는가?)
- storage mapping은 있는데, 왜 memory mapping은 없는가?

3. 컴파일된 contract은 EVM에서 어떻게 보이는가?
- contract은 어떻게 생성되는가?
- constructor는 무엇인가?
- Fallback 함수는 무엇인가?
```
Ethereum VM(EVM) 에서 Solidity와 같은 고급 프로그래밍 언어가 어떻게 동작하는 지 배우는 것은 좋은 투자이다.

```
- Solidity는 완전한 언어가 아니다. (더 좋은 EVM 언어가 나타나고 있다.)
- EVM은 데이터베이스 엔진이다. 어떠한 EVM 언어에서든 smart contract 가 어떻게 동작하는 지 이해하기 위해서는 데이터가 어떻게 구성, 저장, 조작되는 지 이해해야한다.
- Ethereum toolchain은 빠르게 변화하고 있으며, EVM을 아는 것은 멋진 tool 들을 만드는 데 도움이 될 것이다.
- EVM은 programming language design, data structure, cryptography의 교차점에서 놀기 좋은 이유를 제공할 것이다.
```

EVM bytecode가 어떻게 동작하는 지 이해하기 위해서 간단한 Solidity contract 를 이용해 아래와 같은 내용들을 확인할 것이다.  

```
- EVM bytecode의 기초
- 서로 다른 타입(mappings, arrays)이 어떻게 표현되는 가?
- 새로운 contract가 생성될 때 어떤일이 발생하는가?
- 메서드가 호출될 때 어떤일이 발생하는가?
- ABI bridge들과 EVM 언어들의 차이는?
```

최종 목표는 컴파일된 Solidity contract를 이해하는 것이다. 자! 기본적인 EVM bytecode를 읽는 것부터 시작하자! 참고로 EVM Instruction Set 테이블은 유용한 자료이다.


### A Simple Contract

우리가 살펴볼 첫번째 contract는 constructor와 state variable를 가진다.

```
// c1.sol <br>
pragma solidity ^0.4.11;
contract C {
    uint256 a;
    function C() {
      a = 1;
    }
}
```

위와 같은 contract를 solc로 컴파일하면 아래와 같은 어셈블리 코드가 생성된다.

```
$ solc --bin --asm c1.sol
======= c1.sol:C =======
EVM assembly:
    /* "c1.sol":26:94  contract C {... */
  mstore(0x40, 0x60)
    /* "c1.sol":59:92  function C() {... */
  jumpi(tag_1, iszero(callvalue))
  0x0
  dup1
  revert
tag_1:
tag_2:
    /* "c1.sol":84:85  1 */
  0x1
    /* "c1.sol":80:81  a */
  0x0
    /* "c1.sol":80:85  a = 1 */
  dup2
  swap1
  sstore
  pop
    /* "c1.sol":59:92  function C() {... */
tag_3:
    /* "c1.sol":26:94  contract C {... */
tag_4:
  dataSize(sub_0)
  dup1
  dataOffset(sub_0)
  0x0
  codecopy
  0x0
  return
stop
sub_0: assembly {
        /* "c1.sol":26:94  contract C {... */
      mstore(0x40, 0x60)
    tag_1:
      0x0
      dup1
      revert
auxdata: 0xa165627a7a72305820af3193f6fd31031a0e0d2de1ad2c27352b1ce081b4f3c92b5650ca4dd542bb770029
}

Binary:
60606040523415600e57600080fd5b5b60016000819055505b5b60368060266000396000f30060606040525b600080fd00a165627a7a72305820af3193f6fd31031a0e0d2de1ad2c27352b1ce081b4f3c92b5650ca4dd542bb770029
```

참고로 `6060604052…` 는 EVM이 실질적으로 실행하는 bytecode 이다.


### In Baby Steps

위에서 생성된 어셈블리 코드의 절반은 일반적인 Solidity 프로그램들에서 동일하게 사용되는 구문이다. 해당 구문은 나중에 살펴보고 지금은 아래의 코드와 같이 우리의 contract에 한정된 부분만 확인할 것이다. 

```
a = 1
```

위와 같은 Humble storage variable 할당은 6001600081905550 bytecode로 표현된다. 해당 bytecode를 명령어 단위로 하나하나 나누면 아래와 같다.

```
60 01
60 00
81
90
55
50
```

EVM은 기본적으로 top-down 형태로 각 명령어를 실행한다. 
자! 이제 bytecode와 어셈블리 코드가 어떻게 연관되어 있는지 확인하기 위해 “tag_2”의 어셈블리 코드에 bytecode를 주석을 달아보자. (어셈블리 코드와 bytecode에 대한 테이블은 https://ethervm.io 를 참고하면 된다.)

```
tag_2:
  // 60 01
  0x1
  // 60 00
  0x0  
  // 81
  dup2
  // 90
  swap1
  // 55
  sstore
  // 50
  pop
```

어셈블리 코드에 있는 “0x01”은 push(0x01) 명령어를 위한 것이다. 이 명령어는 stack에 “1”을 push한다. 여전히 어렵지만 걱정하지 마라. EVM에서 어셈블리 코드를 한줄 한줄 시뮬레이팅 하는 것은 생각보다 간단하다.


### Simulating The EVM

EVM은 stack machine 이다. 명령어들은 stack 에 있는 arguments 등과 같은 값을 사용하고, 그 결과들을 stack에 push 한다. “add”의 동작에 대해 생각해보자. 
예를 들어 stack에 아래와 같은 두 개의 값이 있다. 

```[1 2]```

이 때 EVM이 “add” 명령어를 수행할 때, 위의 두 값을 더하고, stack에 아래와 같은 결과를 push 한다.

```[3]```

참고로 앞으로 작성되는 내용에서는 아래와 같이 stack을 [ ]으로 contract storaget를 { }으로 표현할 것이다.

```
// The empty stack
stack: []
// Stack with three items. The top item is 3. The bottom item is 1.
stack: [3 2 1]
// Nothing in storage.
store: {}
// The value 0x1 is stored at the position 0x0.
store: { 0x0 => 0x1 }
```

자! 이제 실제 bytecode를 보자. 우리는 EVM에서 6001600081905550 를 시뮬레이팅하고 각 명령어 이후의 machine 상태를 출력할 것이다.

```
// 60 01: pushes 1 onto stack
0x1
  stack: [0x1]
// 60 00: pushes 0 onto stack
0x0
  stack: [0x0 0x1]
// 81: duplicate the second item on the stack
dup2
  stack: [0x1 0x0 0x1]
  // 90: swap the top two items
swap1
  stack: [0x0ㅁ0x1 0x1]
// 55: store the value 0x1 at position 0x0
// This instruction consumes the top 2 items
sstore
  stack: [0x1]
  store: { 0x0 => 0x1 }
// 50: pop (throw away the top item)
pop
  stack: []
  store: { 0x0 => 0x1 }
```

마지막 상태를 보면 stack은 비어있고, storage에는 하나의 값이 저장되어 있다.
위의 내용에서 주목할 부분은 Solidity가 state variable “uint256 a”를 “0x0”에 저장하기로 결정했다는 것이다. 다른 언어들은 state variable를 다른 위치에 저장하도록 완벽하게 결정할 수 있다.
pseudocode 에서 EVM이 6001600081905550 에 대해 수행하는 작업은 아래와 같다.

```
// a = 1
sstore(0x0, 0x1)
```

주의깊게 살펴보면 dup2, swap1, pop이 불필요하다는 것을 알 수 있으며, 불필요한 코드를 제외하면 어셈블리 코드는 더 간단하다.

```
0x1
0x0
sstore
```

위의 3가지 명령어들가지고 시뮬레이팅을 시도하고, 실제로 동일한 machine 상태가 되는 것을 확인할 수 있다.

```
stack: []
store: { 0x0 => 0x1 }
```


### Two Storage Variables

위의 예제에서 동일한 타입의 Extra storage variable 하나를 추가해보자.

```
// c2.sol
pragma solidity ^0.4.11;
contract C {
    uint256 a;
    uint256 b;
    function C() {
      a = 1;
  b = 2;
    }
}
```

위의 Solidity 코드를 컴파일하고, 아래와 같이 “tag_2”를 살펴보자.

```
$ solc --bin --asm c2.sol
// ... more stuff omitted
tag_2:
    /* "c2.sol":99:100  1 */
  0x1
    /* "c2.sol":95:96  a */
  0x0
    /* "c2.sol":95:100  a = 1 */
  dup2
  swap1
  sstore
  pop
    /* "c2.sol":112:113  2 */
  0x2
    /* "c2.sol":108:109  b */
  0x1
    /* "c2.sol":108:113  b = 2 */
  dup2
  swap1
  sstore
  pop
  // a = 1
  sstore(0x0, 0x1)
  // b = 2 
  sstore(0x1, 0x2)
```

pseudocode 에서의 어셈블리 코드는 아래와 같다.

```
 // a = 1
  sstore(0x0, 0x1)
  // b = 2 
  sstore(0x1, 0x2)
```

여기서 우리는 아래와 같이 저장한 두 개의 변수(uint256 a, uint256 b)가 “0x0”, “0x1”에 각각 차례로 위치한다는 점을 알 수 있다.


### Storage Packing

각 storage 슬롯은 32bytes를 저장할 수 있다. 만약 하나의 변수가 16bytes만 필요하더라도 32bytes 모두를 사용하기 때문에 storage가 낭비된다. Solidity는 이러한 낭비를 막고 storage를 효과적으로 사용하기 위해 가능하다면 하나의 stroage 슬롯에 두 개의 작은 데이터를 packing이라는 작업을 수행하여 최적화한다.

```
pragma solidity ^0.4.11;
contract C {
    uint128 a;
    uint128 b;
    function C() {
      a = 1;
      b = 2;
    }
}
```

이를 확인하기 위해 아래와 같이 a, b를 각각 16bytes 로 바꿔보자.

```
$ solc --bin --asm c3.sol
```

위의 코드를 아래의 명령어를 이용하여 컴파일하자.
컴파일 과정을 거쳐 생성된 어셈블리 코드는 아래와 같이 더 복잡해졌다.

```
tag_2:
  // a = 1
  0x1
  0x0
  dup1 
    0x100
  exp
  dup2
  sload
  dup2
  0xffffffffffffffffffffffffffffffff
  mul
  not 
  and
  swap1
  dup4
  0xffffffffffffffffffffffffffffffff
  and
  mul
  or
  swap1
  sstore
  pop
  // b = 2
  0x2
  0x0
  0x10
  0x100
  exp
  dup2
  sload
  dup2
  0xffffffffffffffffffffffffffffffff
  mul
  not
  and
  swap1
  dup4
  0xffffffffffffffffffffffffffffffff
  and
  mul
  or
  swap1
  sstore
  pop
```

위의 어셈블리 코드는 두 개의 변수가 하나의 storage 위치(“0x0”)에 아래와 같이 packing을 수행한다.

```
[         b         ][         a         ]
[16 bytes / 128 bits][16 bytes / 128 bits]
```

Packing을 하는 이유는 아래와 같이 storage 사용하는 작업은 gas가 가장 많이 들기 때문이다.

- sstore는 새로운 position을 사용하기 위해 20000 gas
- sstore는 존재하는 position에 덮어쓰기 위해 5000 gas
- sload는 500 gas
- 대부분의 명령어는 3~10 gas

같은 storage position을 사용하기 위해, Solidity는 packing 작업을 통해 “uint128 b”를 저장하는데 있어 20000 gas 대신에 5000 gas를 지불하여, 15000 gas를 절약한다.


### More Optimization

a와 b를 저장하는 sstore 명령어를 두개로 분리하지 않고, 메모리에서 두 개의 128bits 값을 packing하고 하나의 sstore 명령어만을 사용하면 추가적으로 5000 gas를 절약할 수 있다. 이러한 작업은 아래와 같이 컴파일 시 optimize flag를 이용하여 Solidity가 최적화하도록 만들수 있다.

```$ solc --bin --asm --optimize c3.sol```

위의 명령어에 의해 생성된 어셈블리 코드는 앞서 의도한 바와 같이 하나의 sload, sstore 가 사용된다

```
tag_2:
    /* "c3.sol":95:96  a */
  0x0
    /* "c3.sol":95:100  a = 1 */
  dup1
  sload
     /* "c3.sol":108:113  b = 2 */
  0x200000000000000000000000000000000
 not(sub(exp(0x2, 0x80), 0x1))
    /* "c3.sol":95:100  a = 1 */
  swap1
  swap2
  and
    /* "c3.sol":99:100  1 */
  0x1
    /* "c3.sol":95:100  a = 1 */
  or
  sub(exp(0x2, 0x80), 0x1)
    /* "c3.sol":108:113  b = 2 */
  and
  or
  swap1
  sstore
```

위의 어셈블리 코드에 대한 bytecode는 아래와 같다.

```
600080547002000000000000000000000000000000006001608060020a03199091166001176001608060020a0316179055
```

그리고 각 명령어에 bytecode를 맞춰보면 아래와 같다.

```
// push 0x0
60 00
// dup1
80
// sload
54
// push17 push the the next 17 bytes as a 32 bytes number
70 02 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
/* not(sub(exp(0x2, 0x80), 0x1)) */
// push 0x1
60 01
// push 0x80 (32)
60 80
// push 0x80 (2)
60 02
// exp
0a
// sub
03
// not
19
// swap1
90
// swap2
91
// and
16
// push 0x1
60 01
// or
17
/* sub(exp(0x2, 0x80), 0x1) */
// push 0x1
60 01
// push 0x80
60 80
// push 0x02
60 02
// exp
0a
// sub
03
// and
16
// or
17
// swap1
90
// sstore
55
```

그리고 위의 어셈블리 코드 안에는 4개의 magic value가 있다.

- 0x1 (16 bytes), using lower 16 bytes
```
// Represented as 0x01 in bytecode
16:32 0x00000000000000000000000000000000
00:16 0x00000000000000000000000000000001
```
- 0x2 (16 bytes), using higher 16bytes
```
// Represented as 0x200000000000000000000000000000000 in bytecode
16:32 0x00000000000000000000000000000002
00:16 0x00000000000000000000000000000000
```
- not(sub(exp(0x2, 0x80), 0x1))
```
// Bitmask for the upper 16 bytes
16:32 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
00:16 0x00000000000000000000000000000000
```
- sub(exp(0x2, 0x80), 0x1)
```
// Bitmask for the lower 16 bytes
16:32 0x00000000000000000000000000000000 
00:16 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```
코드는 이 값을 사용하여 “bits-shuffling”을 하여 원하는 결과에 도달한다.
```
16:32 0x00000000000000000000000000000002 
00:16 0x00000000000000000000000000000001
```

마침내, 32bytes 값은 “0x0”에 저장되었다.


### Gas Usage

60008054700**200000000000000000000000000000000**6001608060020a03199091166001176001608060020a0316179055

위의 bytecode를 보면 **“0x200000000000000000000000000000000”** 이 bytecode 에 내장되어 있다. 그러나 컴파일러는 명령어 **“exp(0x2, 0x81)”**을 이용하여 값을 계산할 수 있고 bytecode 시퀀스가 더 짧아진다. 하지만 bytecode는 **“exp(0x2, 0x81)”** 보다 더 싼 **“0x200000000000000000000000000000000”** 으로 표현된다. 아래와 같이 transaction을 위한 Gas 비용을 살펴보자.

- transaction을 위한 zero byte의 code 또는 data는 4 gas
- transaction을 위한 non-zero byte의 code 또는 data는 68 gas

위의 기준을 바탕으로 **“0x200000000000000000000000000000000”**와 **“exp(0x2, 0x81)”**에 드는 비용이 얼마인지 비교해보자.

- 0x200000000000000000000000000000000
 (1 * 68) + (16 * 4) = **196 gas**

- exp(0x2, 0x81) = 608160020a
 5 * 68 = **340 gas**

결과적으로 zero-byte가 많은 긴 bytecode 의 gas 소모량이 적다.


### Summary

EVM 컴파일러는 bytecode 크기, 속도, 메모리를 효과적으로 사용하기 위한 최적화가 존재하지 않지만 gas 사용량을 최적화하며 이것은 Ehtereum 블록체인이 효율적으로 수행할 수 있는 계산을 인센티브로 하는 간접 계층이다. 

- **EVM은 256 bit machine** 이다. 32bytes 단위로 데이터를 조작하는 것이 가장 자연스럽다.
- 영구 저장장치는 상당히 비싸다. (gas가 많이 든다)
- **Solidity 컴파일러는 gas 사용량을 최소화**하기 위해 흥미로운 선택을 한다.

Gas 비용은 다소 자의적으로 설정되어 있으며, 향후에 변경될 수 있다. 비용 변화에 따라, 컴파일러들 또한 다른 선택을 할것이다. 
