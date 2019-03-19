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
₩₩₩



### Parsecs Upon Parsecs of Tape

