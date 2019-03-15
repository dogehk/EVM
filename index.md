## The Ethereum Virtual Machine

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







**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/dogehk/EVM/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
