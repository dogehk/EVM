해당 글은 Howard님이 2017/08/`3 에 작성한 [Diving Into The Ethereum VM Part 2](https://medium.com/@hayeah/diving-into-the-ethereum-vm-part-2-storage-layout-bc5349cb11b7) 을 번역한 글입니다. 변역 과정에서 자연스러운 문장을 위해 다소 변경 부분이 존재할 수 있습니다.

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

![Alt text](./images/TuringMachine.jpg "Turing Machine. Source : http://raganwald.com/")
