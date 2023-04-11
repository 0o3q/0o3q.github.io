---
layout: post
title: "블록체인과 비트코인"
tags: 해시 블록체인 비트코인
---

## 블록체인의 개념과 구조

블록체인은 단어 그대로 해석해보면 블록이 서로 연결되어 있다라는 의미이다. 
여기서 블록이란 거래(or 트랜잭션) 기록을 담고있는 장부의 한 페이지라고 생각하면 이해가 쉽다.
즉 블록에는 거래 당사자들의 모든 거래 내용을 담고 있으며 일정한 조건을 만족하면 하나의 블록을 완성시키고 다음 블록에 새로운 거래 내용을 기록하게 된다. 
여기서 일정한 조건이란 나중에 설명할 블록 생성을 위한 합의 알고리즘이다.

다음 블록에는 이전 블록들의 모든 정보를 압축하여 담아 두는데, 이를 위해 SHA-256해시를 활용한다.
이러한 블록들은 거래가 참여하는 모든 당사자들에게 공유됨으로써 거래에 대한 투명성을 높이고 이전 블록들의 압축된 정보를 다음 블록에 기록함으로써 위조나 변조가 매우 힘들도록 한 것이다.

블록체인을 한 문장으로 정의하면 다음과 같이 요약할 수 있다.

---

“블록”이라고 하는 거래 기록을 담고있는 데이터들이 거래 참여자들의 합의에 생선된 체인 형태의 연결고리를 가진 형태로 구성되어 모든 거래 참여자들에 배포되며, 누구나 기록된 거래 내용의 결과를 열람할 수 있지만, 어느 누구도 임의로 수정할 수 없도록 만든 분산 컴퓨팅 기반의 데이터 위변조 방지 기술이다.

---

블록체인은 일반적으로 다음과 같은 구조를 띈다.

![Image blockchain_structure]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/blockchain_structure.png)

블록체인은 블록 1에서부터 거래를 기록하게 되며 특정한 조건인 합의 알고리즘을 통해 블록 1에 더이상 거래를 기록할 수 없도록 블록을 동결하고 완성시킨다.
블록 1에 기록된 정보를 대표하는 해시값 Hash1을 생성하고 Hash1은 다음 블록인 블록 2에 기록한다. 블록 2도 특정한 조건에 만족하면 블록 1과 같은 과정으로 거치고 그 다음 블록인 블록 3에 거래를 기록하는 방식으로 진행된다. 이런식으로 구성되는 블록체인은 거래에 참여하는 모든 당사자들에게 배포된다.

블록체인에서 첫 번째 블록인 블록 1을 “제네시스 블록”이라 부르고 이 블록의 정보가 거짓이라면 블록체인의 전체 정보가 거짓이 되기 때문에 제네시스 블록에는 반드시 참 값을 기록해야 한다.

블록체인을 구성하는 블록의 일반적인 구조는 다음과 같다.

![Image block_structure]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/block_structure.png)

블록은 블록헤더(block header)와 블록몸체(block body)로 이루어져 있다. 
블록몸체에는 거래를 기록하며, 블록헤더에는 이전 블록헤더의 해시값, 현재 블록에 기록되어 있는 전체 거래에 대한 해시값 그리고 기타 정보들이 기록되어 있다. 블록헤더의 기타 정보 부분은 블록체인의 종류에 따라 기록되는 데이터가 다르다.

블록헤더에 기록되어 있는 해시값은 모두 SHA-256을 이용한다. 블록헤더에 기록된 현 블록의 전체 트랜잭션에 대한 해시값은 머클트리(Merkle Tree)라는 방법으로 계산되는데, 그림에서 보는 것처럼 피라미드구조와 같이 2개의 트랜잭션에 대한 해시값을 짝지어 이 2개의 해시값에 대한 해시값을 계산하는 방법으로 피라미드 꼭대기에 있는 1개의 해시값으로 만드는 것이다.
피라미드 꼭대기에 있는 해시값을 머클루트(Merkle Root)라 부르며, 이 머클루트가 블록헤더의 전체 트랜잭션 해시값이 되는 것이다.

따라서 블록헤더에는 현재 블록의 전체 거래 기록들의 압축된 정보가 기록되어 있는 셈이다.
이 블록헤더에 대한 해시값은 나중에 다음 블록의 블록헤더에 이전 블록 해시값 부분에 기록된다.

물론 블록헤더에 기록되는 머클루트는 블록이 완성되기까지 거래기록이 추가되면 계속 변하는 값이다. 다음 블록에 기록할 현재 블록헤더에 대한 해시값은 블록에 더 이상 거래를 기록할 수 없도록 한 상태에서 만들어진다.

블록몸체에 기록되는 거래 즉 트랜잭션 데이터는 거래 정보와 부가 데이터를 저장할 수 있는 공간으로 되어있다.

지금까지 설명한 블록체인의 구조는 블록체인의 종류에 상관없이 거의 공통적인 내용이며, 블록이 만들어지는 조건인 합의 알고리즘이나 블록의 크기등이 블록체인의 종류에 따라 달라진다.

## 블록체인과 작업증명

블록체인에선 블록은 거래 참여자들의 합의에 의해 생성되고 배표된다고 언급했다. 합의 알고리즘은 분산 네트워크 상에 존재하는 서로 신뢰할 수 없는 정보들에 대해 수학적으로 계산된 값을 특정 절차에 따라 상호 검증하여 신뢰 가능하도록 보장하는 알고리즘이다.
블록체인의 연결 구조와 합의 알고리즘이라는 메커니즘을 통해 블록체인에 담긴 정보들의 위조나 변조가 매우 어려워진다.

블록체인에서 활용되는 합의 알고리즘은 다양하게 존재하는데 대표적으로 작업증명(Proof Of Work), 지분증명(Proof Of Stake), PBFT(Practical Byzantine Fault Tolerance), RAFT 등이 있다.

우선 해시와 매우 밀접한 작업 증명에 대해서만 알아본다.

작업증명은 주어진 해시값보다 작아지도록 SHA-256에 입력되는 값을 찾아내는 것이다.
작업증명의 난이도는 주어진 해시값에 따라 달라진다.

---

문제 1

SHA-256(value) ≤ 000c007…3c240b36c312

---

위 문제 1을 만족하는 value를 찾아낸느 것이 작업증명의 메커니즙이다. 작업증명을 더 쉽게 이해할 수 있도록 다음의 문제를 생각해보자

---

“Attack at 9PM!”이라는 문자에 대한 SHA-256해시값은 다음과 같다.

SHA-256(”Attack at 9PM!”) = c53ae0b1db6f94ce4177112…9f5a19f47d473a785dc97afc

---

이제 “Attack at 9PM!” 문장 뒤에 숫자 0을 더해서 SHA-256 해시값을 계산해보면 다음과 같은 결과가 나온다.

---

SHA-256(”Attack at 9PM!0”) = 8eb6977a7765fda7cf7d6…20ede2b3c371ab6c9adc41a0

---

여기서 문제를 제시한다.

---

문제 2

“Attack at 9PM!”에 어떤 숫자를 추가하여 구한 SHA-256 해시값이 00으로 시작되도록하는 어떤 숫자를 구하라

---

문제 2를 풀기 위해서는 “Attack at 9PM!”에 0을 추가하여 SHA-256 해시값을 구하고 을 추가하여 SHA-256 해시값을 구하는식으로 부르트포싱하여 구한 해시값이 00으로 시작되는 그 숫자를 찾으면 된다.

가장 최초로 나타나는 값은 240이다.

문제 2와 같은 유형의 문제를 해시캐시(hashcash)라고 부른다. 해시캐시 문제의 답을 구하려면 그냥 무식하게 부르트포싱하는 방법뿐이다. 부르트포싱하며 원래 메세지에 추가하는 값을 nounce라고 한다. 즉 해시캐시 문제는 주어진 해시값고 같아지는 nounce 값을 찾는 것이다.

문제 2에서 주어진 문제가 00으로 시작하는게 아닌 000으로 시작하는 해시값이면 난이도는 더욱 올라가고 nounce는 6541이 된다. 따라서 0의 개수가 많아질 수록 해시캐시 문제의 난이도는 증가한다.

작업증명은 해시캐시 문제를 푸는 원리와 동일하다

다음은 문장 “Attack at 9PM!”에 대한 해시캐시를 구하는 파이썬 코드이다.

```python
from hashlib import sha256 as sha

def hashcash(msg, difficulty):
    nounce = 0
    print("+++ Start")
    while True:
        target = "%s%d" %(msg, nounce)
        ret = sha(target.encode()).hexdigest()

        if ret[:difficulty] == '0'*difficulty:
            print("++++ Bingo")
            print("--->", ret)
            print("---> NOUNCE = %d" %nounce)
            break

        nounce += 1

def main():
    msg = "Attack at 9PM!"
    difficulty = 1
    hashcash(msg, difficulty)

if __name__ == "__main__":
    main()
```

difficulty의 개수에 따라 0의 개수가 정해지며 difficulty가 커지면 시간도 그에 맞게 커진다.

비트코인에서 사용되는 작업증명은 비트코인의 작업증명에서 자세히 다룬다.

## 작업증명의 효과

앞에서 제시한 해시캐시는 난이도가 매우 낮아서 일반 PC 1대로도 순식간에 nounce를 구할 수 있다.하지만 비트코인이나 이더리움같은 블록체인 네트워크에서는 네크워크에 참여하는 모든 노드(실제로는 블록을 생성하기 위해 참여한 노드)의컴퓨팅 파워를 총 동원하여 일정한 시간동안 계산해야만 nounce를 구할 수 있을 정도로 해시캐시 난이도를 조정한다. 

비트코인의 경우 참여하는 모든 노드의 컴퓨팅 파워를 동원하여 10분 정도 걸리는 해시캐시 난이도로 설정한다.

실제 작업증명의 메커니즘은 다음과 같다.

1. 거래 정보가 담긴 1번 블록의 정보를 이용해 해시캐시 문제를 제시한다.
2. 제시된 해시캐시 문제는 블록체인에 참여한 모든 노드의 컴퓨팅 파워로 10분정도 걸려야 풀 수 있는 난이도로 설정되어 있다.
3. 특정 노드에서 해시캐시 문제의 답을 구했다면 1번 블록에 정답을 기록하고 더 이상의 거래 기록을 담지 못하게 동결하고 이 블록을 모든 참여 노드에 전달한다.
4. 해시캐시 문제의 정답을 구하는 것은 어렵지만 답을 구하면 그 답이 정답인지 검증하는 것은 매우 쉽다. 따라서 블록을 전달받은 모든 블록에 포함된 해시캐시 정답을 검증하고 정말 정답이면 블록을 수용하고 그렇지 않으면 블록을 폐기한다.
5. 새로운 2번 블록에 거래를 기록한다.
6. 2번 이후 블록에 대해 1~5 과정을 반복한다.

이제 다음과 같은 상황을 가정한다.

100개의 노드(각 노드는 동일한 컴퓨팅 파워를 가지고 있다고 가정한다.)가 블록체인 네트워크에 참여하고 있고, 현재 n번째 블록까지 생성된 상태이다. 100개의 노드 중 10개의 노드가 n+1번째 블록의 거래 내용을 조작한다고 할 때, 작업증명의 효과로 다음과 같은 상황이 된다.

![Image pow_effect]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/pow_effect.png)

- 조작된 n+1번째 블록 내용을 기반으로 100개의 노드가 10분간 풀어야할 해시캐시 문제를 10개 노드가 해결해야한다.
- 그동안 정상적인 n+1번째 블록 내용을 기반으로 100개의 노드가 10분간 풀어야할 해시캐시 문제를 90개의 노드가 해결하고 있는 중이다.
- 확률적으로 정상적인 n+1번째 블록에 대한 해시캐시 문제가 훨씬 빨리 해결될 것이고, n+1번째 블록을 생성한 후 n+2번째 블록에 대한 해시캐시 문제를 90개의 노드가 해결하고 있는 상황이 된다.
- 이제 10개의 노드는 n+1번째 블록뿐만 아니라 n+2번째의 블록에 대한 해시캐시 문제를 해결해야 되는 지경에 이른다.

이런 원리를 통해 10개의 노드는 블록의 내용을 위변조하는 것은 거의 불가능에 가깝게 된다.
따라서 블록체인에서 작업증명의 효과는 블록체인 구조에서 기인되는 위변조 조작의 어려움에 더해 더욱 위변조가 어려워지도록 하는 효과를 가진다.

이럼에도 불구라고 1개의 노드가 조작된 블록에 해시캐시 문제를 해결하고 모든 노드에 배표하게 된다면, 조작된 블로깅 포함된 브록체인의 기링가 정상적인 블록체인의 길이보다 짧을 것이다.

![Image pow_effect2]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/pow_effect2.png)

블록체인 시스템은 2개이상의 블록체인이 배포될때 길이가 딘 블록체인의 채캑을 원칙으로 하고 있다.

## 비트코인의 블록구조

비트코인은 브록체인 기술을 활용한 최초의 분산원장 기술이자 가장 잘 알려진 암호화폐 시스템이다. 비트코인을 기본적으로 이해하고 있다면 다른 블록체인 시스템에 대해 보다 쉽게 접근할 수 있다.

비트코인의 즐록 구조는 블록체인의 일반적인 구종와 동일하다. 실제 비트코인의 블록에는 다음의 그림과 같은 정보들로 구성되어 있다.

![Image bitblock_structure]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/bitblock_structure.png)

위 그림처럼 매직넘버와 블록 크기를 나타내는 필드를 제외하면 비트코인 블록은 블록헤더와 블록몸체로 구성되어 있는 것을 볼 수 있다. 블록헤더는 총 6개의 필드로 구성되어 있고, 블록몸체에는 거래 개수와 거래 정보가 담겨있다.

블록 몸체를 이루는 값들에 대해 간단히 살펴보면 Trancsaction counter는 블록몸체에 기록된 거래 개수이다. 거래 정보가 블록봅체에 추가될 때마다 Transaction counter는 1증가한다.

블록몸체에서 Coinbase transaction은 새로운 블록이 생성될 때 블록몸체에 최초로 기록되는 거래 정보로. 생성 거래로 부르기도 하며 초기에는 무의미한 값들로 구성되어 있다. 하지만 작업증명이 성공하게 되면 새로 발행되는 비트코인을 성공한 노드의 비트코인 주소로 입금하는 거래 정보로 변경된다.

블록헤더의 6개 필드 중 첫 3개는 앞에서 설명했으니 Time 필드부터 살펴본다.
Time 필드는 현재 시간을 기록하는 타임스탬프 역할을 하는데, 이 값은 몇 초마다 갱신되며 블록이 완성될 때까지 지속적으로 변한다.

Bits는 작업증명을 위한 대상값(Target)을 저장하는 부분인데 실제로는 Bits에 저장된 값을 특정 공식에 적용하여 대상값을 산출해내며, 이 대상값보다 블록헤더의 해시값이 작아지게 되는 nounce를 구하면 작업증명에 성공하게 된다. 이에 대해서는 실제 비트코인 블록헤더를 이용하여 작업증명 메커니즘을 설명할 때 알아본다.

Bits는 비트코인 네트워크에 참여하고 있는 노드의 수에 따라 값이 변하는데, 앞에서 설명한 바와 같이 모든 참여자 노드의 컴퓨팅 파워를 이용해 해시캐시 문제를 10분 정도 걸려야 풀 수 있도록 조정된다.

다음은 실제 비트코인의 125,552번째 블록헤더의 내용이다.

![Image 12552bit]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/12552bit.png)

위 그림에서 각 필드의 값들은 모두 리클 엔디안 방식으로 메모리에 저장되어 있는 값을 보여주고 있다.

인텔 x86 계열의 CPU는 리틀 엔디안 방식으로 메모리에 데이터를 저장한다.
리틀 엔디안이란 하뉘 주소의 메모리에 낮은 자리의 숫자를 기록하는 방식이다.
16진수 0x1b0404cb를 리틀 엔디안으로 메모리에 저장하면 다음과 같다.

![Image little]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/little.png)

따라서 실제 위 그림의 블록헤더에 저장되어 있는 Bits의 값은 0x1a449b2f, Nounce의 값은 0x9546a142이다.

## 비트코인의 작업증명

비트코인의 작업증명은 블록헤더 6개 필드에 대한 해시캐시를 구하는 문제이다. 해시캐시의 난이도는 Bits 필드에 저장된 Target값으로 결정된다.

---

비트코인 작업증명

SHA-256(블록헤더 6개 필드) ≤ Target

---

Targt값은 Bits에 저장된 값을 특정 변환 공식을 적용하여 얻어진다.
비트코인의 난이도 설정값인 4바이트 크기의 Bits를 Target값으로 변환하는 공식은 다음과 같다.

---

Bits의 값이 0x1a44b9f2인 경우,
1. Bits의 앞 2자리와 쥐 6자리를 다음과 같이 분리함
2. 1a 44b9f2
3. Target = 0x44b9f2 * 2**(8*(0x1a-3))

---

다음은 리틀 엔디안 으로 저장된 Bits를 Target으로 변환하는 파이썬 코드다.

```python
from hashlib import sha256 as sha
import codecs

#인자 bits를 실제 16진수 값으로 리턴
def decodeBitcoinVal(bits):
    decode_hex = codecs.getdecoder('hex_codec')
    binn = decode_hex(bits)[0]
    ret = codecs.encode(binn[::-1], 'hex_codec')
    return ret

def getTarget(bits):
    bits = decodeBitcoinVal(bits)
    bits = int(bits, 16)
    print("Bits = %x" %bits)
    bit1 = bits >> 4*6
    base = bits & 0x00ffffff

    sft = (bit1 - 0x3)*8
    target = base << sft
    print("Target = %x" %target)

Bits = "f2b9441a"
getTarget(Bits)
```

실제 비트코인의 125,552번째의 Bits값 사용

```python
def decodeBitcoinVal(bits):
    decode_hex = codecs.getdecoder('hex_codec')
    binn = decode_hex(bits)[0]
    ret = codecs.encode(binn[::-1], 'hex_codec')
    return ret
```

위 코드에서 리틀 엔디안으로 표현된 16진수값을 받아와 실제 16진수 순서로 변환한 후 정수 값으로 리턴한다.

```python
def getTarget(bits):
    bits = decodeBitcoinVal(bits)
    bits = int(bits, 16)
    print("Bits = %x" %bits)
    bit1 = bits >> 4*6
    base = bits & 0x00ffffff

    sft = (bit1 - 0x3)*8
    target = base << sft
    print("Target = %x" %target)

```

Bits를 Target값으로 변환하는 공식 구현

1비트를 오른쪽으로 쉬프트하면 원래의 값에 2를 곱한 것과 동일하다. 반대로 1비트를 왼쪽으로 쉬프트하면 2를 나눈 것과 동일하다.

16진수는 4비트로 표현되므로 A>>4와 A<<4는 각각 원래의 값 A에 대해 16진수 자리수를 한자리 내리거나 또는 한자리 올린다는 의미와 같다.

코드를 실행하면 Target이 다음과 같이 출력된다.

---

Target = 44b9f20000000000000000000000000000000000000000000000

---

비트코인에서 Target은 해시캐시 문제를 풀기 위한 대상이라고 했다. 비트코인 블록헤더에 기록된 6개 필드에 대한 SHA-256 해시값이 이 값보다 작게 되는 nounce값을 찾는 것이다.

SHA-256 해시는 32바이트로 표현되므로, 위 Target을 32바이트로 표현하면 다음과 같은 값이 된다.

---

Target = 00000000000044b9f20000000000000000000000000000000000000000000000

---

따라서 다음의 식을 만족하는 Nounce 값을 계산해야 비트코인의 125.552번째 블록이 완성된다.

---

SHA-256(Version+HashPrevBlock+HashMerkleRoot+Time+Bits+Nounce) ≤ 00000000000044b9f20000000000000000000000000000000000000000000000

---

다음 코드는 비트코인의 125,532번째 블록에 대해 Nounce값을 검증하는 코드이다. 비트코인의 125,532번째 블록은 이미 오래전에 만들어진 블록이므로 기록된 Nounce값이 위 해시캐시 문제를 만족하는 값일 것이다

```python
from hashlib import sha256 as sha
import codecs

#인자 bits를 실제 16진수 값으로 리턴
def decodeBitcoinVal(bits):
    decode_hex = codecs.getdecoder('hex_codec')
    binn = decode_hex(bits)[0]
    ret = codecs.encode(binn[::-1], 'hex_codec')
    return ret

def getTarget(bits):
    bits = decodeBitcoinVal(bits)
    bits = int(bits, 16)
    bit1 = bits >> 4*6
    base = bits & 0x00ffffff

    sft = (bit1 - 0x3)*8
    target = base << sft

    return target

# 아래에는 2011년 5월에 생성된 비트코인 125,552번째 블록에 대한 작업 증명이 성공했음을 검증하는 코드이다.

def validatePoW(header):
    block_version = header[0]
    hashPrecBlock = header[1]
    hashMerkleRoot = header[2]
    Time = header[3]
    Bits = header[4]
    nounce = header[5]

    #블록 헤더의 모든 값을 더하고 이름 바이트 객체로 변경
    decode_hex = codecs.getdecoder('hex_codec')
    header_hex = block_version+hashPrecBlock+hashMerkleRoot+Time+Bits+nounce
    header_bin = decode_hex(header_hex)[0]

    # 실제 비트코인에서는 블록헤더의 SHA-256 해시에 대한 SHA256해시를 이용해서 작업증명을 한다.
    hash = sha(header_bin).digest()
    hash = sha(hash).digest()
    PoW = codecs.encode(hash[::-1], 'hex_codec')

    #헤더의 Bits 값을 이요해 실제 target 값 추출
    target = getTarget(Bits)
    target = str(hex(target))
    target = '0'*(66-len(target)) + target[2:]

    print("target\t=", target)
    print("BlockHash\t=", PoW.decode())

    # 작업 증명이 성공한 건지 체크
    if int(PoW, 16) <= int(target, 16):
        print("+++ Accept this Block")
    else:
        print("--- Reject this Block")

def main():
    block_version = "01000000"
    hashPrevBlock = "81cd02ab7e569e8bcd9317e2fe99f2de44d49ab2b8851ba4a308000000000000"
    hashMerkleRoot = "e320b6c2fffc8d750423db8b1eb942ae710e951ed797f7affc8892b0f1fc122b"
    Time = "c7f5d74d"
    Bits = "f2b9441a"
    nounce = "42a14695"
    header = [block_version, hashPrevBlock, hashMerkleRoot, Time, Bits, nounce]
    validatePoW(header)

main()
```

실행 결과

`target  = 00000000000044b9f20000000000000000000000000000000000000000000000
BlockHash       = 00000000000000001e8d6829a8a21adc5d38d0a473b144b6765798e61f98bd1d
+++ Accept this Block`

![Image pow_result]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/pow_result.png)

비트코인 블록 검색 사이트 아무거나 들어가면 일치하는 것을 볼 수 있다.

---

```python
import codecs

#인자 bits를 실제 16진수 값으로 리턴
def changeEndian(bits):
    decode_hex = codecs.getdecoder('hex_codec')
    binn = decode_hex(bits)[0]
    ret = codecs.encode(binn[::-1], 'hex_codec')
    print(ret)

Bits = "2b12fcf1b09288fcaff797d71e950e71ae42b91e8bdb2304758dfcffc2b620e3"
changeEndian(Bits)
```

쓸모 있을 지 모르지만 엔디안 바꾸는 코드이다.

## 비트코인 주소 생성하기

모든 블록체인 시스템은 은행계좌번호와 비슷한 역할을 하는 블록체인 주소라는 것이 있는데, 비트코인에서 이 주소를 비트코인 주소라고 부른다.
A라는 사람이 B라는 사람에게 비트코인을 전송하려면, A의 비트코인 주소에 담겨있는 비트코인을 B의 비트코인 주소로 보내주면 된다.

비트코인 주소는 일반적인 경우 1, 다중 서명이 가능한 경우 3으로 시작하는 34자 길이로 되어있다.

일반적인 비트코인 주소를 생성하는 방법에 대해 알아보자
비트코인을 포함한 모든 블록체인 시스템에는 거래를 생성할 때 전자서명 알고리즘을 활용하고 있다.

예를 들어 비트코인을 거래할 때 비트코인을 전송하는 사람이 자신의 개인키로 서명을 하고 거래 정보와 자신의 공개키를 상대방의 비트코인 주소로 전달하면 비트코인을 받는 사람은 동봉된 공개키가 그 사람의 공개키로 서명을 확인하고 검증되면 거래가 성사된다.

비트코인에 활용되는 공개키 시스템은 비트코인 주소를 만드는 것에도 활용된다. 비트코인 주소는 이 비트코인 주소 소유자의 공개키를 기반으로 만들어진다. 비트코인에 적용되는 전자 서명 알고리즘은 ECDSA이다.

![Image bitaddr]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/bitaddr.png)

1. 개인키를 이용해 ECDSA 공개키를 얻음
2. EDCSA 공개키의 앞부분에 “0x04”를 추가한다.
3. 2단계에서 얻은 값의 SHA-256 해시값을 얻고, 이 해시값에 RIPEMD-160을 적용한 값을 얻는다.
4. 3단계에서 얻은 값에 앞에 “0x00”을 추가한다.
5. 4단계에서 얻은 값의 더블 SHA-256 해시값을 얻는다.
6. 5단계에서 얻은 값의 첫 4바이트를 체크섬으로 둔다.
7. 4단계에서 얻은 Base58 인코딩을 적용한 값을 비트코인  주소로 한다.

이 순서를 순서도로 보면

![Image bitaddr2]({{site.url}}/images/2023-04-13-blockchain_wih_bitcoin/bitaddr2.png)

여기서 등장하는  RIPEMD-160 은 1996년 벨기에의 루벤 카톨릭 대학의 COSIC 연구그룹에서 MD4를 기반으로 개발한 해시 알고리즘이다.
RIPEMD-160은 임의의 입력값에 대해 160비트 크기의 해시값을 출력하며, 32비트 연산에 최적화 되어있다.

비트코인 주소 생성 메커니즘을 파이썬으로 구현한 코드이다.
```python
# pip install base58check
# pip install ecdsa
import os
import hashlib
from hashlib import sha256 as sha
from base58check import b58encode
import ecdsa

def ripemd160(x):
    ret = hashlib.new("ripemd160")
    ret.update(x)
    
    return ret

def generateBitcoinAdress():
    # 개인키 생성
    privkey = os.urandom(32)
    fullkey = "80" + privkey.hex()

    a = bytes.fromhex(fullkey)
    sha_a = sha(a).digest()
    sha_b = sha(sha_a).hexdigest()
    c = bytes.fromhex(fullkey+sha_b[:8])

    # WIF = Wallet Import Format -> 비트코인 거래를 위한 약식 개인키
    WIF = b58encode(c)

    # 1단계 ECDSA 공개키 획득
    signing_key = ecdsa.SigningKey.from_string(privkey, curve=ecdsa.SECP256k1)
    verifying_key = signing_key.get_verifying_key()
    pubkey = (verifying_key.to_string()).hex()

    # 2단계
    pubkey = "04" + pubkey

    # 3단계
    pub_sha = sha(bytes.fromhex(pubkey)).digest()
    encPubkey = ripemd160(pub_sha).digest()

    # 4단계
    encPubkey = b"\x00" + encPubkey

    # 5단계
    chunck = sha(sha(encPubkey).digest()).digest()

    # 6단계
    checksum = chunck[:4]

    # 7단계
    hex_address = encPubkey + checksum

    # 8단계
    bitcoinAdress = b58encode(hex_address)

    # WIF와 생성된 비트코인 주소 출력
    print("+++WIF = ", WIF.decode())
    print("+++Bitcoin Adress = ", bitcoinAdress.decode())

generateBitcoinAdress()
```