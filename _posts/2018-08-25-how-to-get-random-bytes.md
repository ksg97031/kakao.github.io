---
layout: post
title: '리눅스에서 난수는 어떻게 생성할까??'
author: sg.kim
date: 2018-08-25 02:00
tags: [security]
image: /files/covers/how-to-get-random.jpg
---

## /dev/urandom

예전부터 난수 추출은 무조건 **/dev/urandom**에서 가져왔다.
그러다가 우연히 난수를 생성하는 코드를 작성할 일이 있었고, 그 일을 하는 와중에 난수라는 게 생각보다 비현실적이고 불가능하다는 생각이 들었다.
그런데도 도대체 urandom은 어떻게 신뢰할 수 있는 난수 값을 반환하는 것인지, 난생 사용해보지 않은 /dev/random은 어떤 친구인지 궁금해졌다.


## 어떻게 난수를 생성할까?

### PRNG(pseudorandom number generator)

![random](https://t1.daumcdn.net/cfile/tistory/99B3C34F5A435C1222)

리눅스 커널에는 **Linux PRNG**(pseudorandom number generator)라는 유사난수 생성기가 있다. 여기서 유사난수란 진짜 난수(truly random)가 아니라 
PRNG's seed 즉 예전 우리가 random 함수를 사용할 때 현재 시각으로 설정하곤 했던 그 seed 값에 의해 생성된 난수다. 유닉스 계열 운영체제에 존재하는 /dev/random, /dev/urandom 익숙한 장치 파일이 바로 유사난수 발생기 역할을 한다.    


### Entropy 

![RNG](https://image.slidesharecdn.com/slideshare-linuxrng-160328130230/95/slideshare-linux-random-number-generator-2-638.jpg?cb=1459170255)

컴퓨터 과학에서 **Entropy**는 TCP Sequence 같이 난수가 필요한 OS, Application에서 사용하기 위해 임의에 무작위 데이터를 수집하는걸 의미한다.
사용자의 마우스, 키보드 I/O, Events, Interrupts, IDE 같은 다양한 **Entropy Sources**를 통해 무작위에 데이터를 긁어다가 **input pool**에 저장한다.

### Entropy Pool

PRNG에는 3개의 **Entropy Pool**이 있다. 
```c
static __u32 input_pool_data[INPUT_POOL_WORDS];
static __u32 blocking_pool_data[OUTPUT_POOL_WORDS];
static __u32 nonblocking_pool_data[OUTPUT_POOL_WORDS];
```
위에서 말한 input pool 그리고 blocking pool, nonblocking pool이 있다.
input pool에 저장된 데이터는 PRNG 내부 로직에 맞게 재배치, 해시화, XOR 등 암호학적으로 안전하게 데이터를 재생성된 후 blocking, nonblocking pool로 추출된다.
위 같은 방법으로 생성된 blocking pool과 nonblocking pool에서 데이터를 가져다 사용한게 바로 /dev/random과 /dev/urandom이다.

### /dev/random vs /dev/urandom
그러면 도대체 두 장치 파일에 차이점은 무엇일까?  

blocking pool를 사용하는 /dev/random은 저장된 entropy를 순수히 사용한다. 그래서 만약 input pool에 저장된 entropy 데이터에 개수가 부족하면 무한히 행에 걸리고 만다. 아래 소스 같이 **entropy_count** 변수를 이용해서 가져올 데이터가 있는지 확인한다.
```c
if (unlikely(entropy_count < 0)) {
  pr_warn("random: negative entropy count: pool %s count %d\n",
    r->name, entropy_count);
  WARN_ON(1);
  entropy_count = 0;
}
```

nonblocking pool을 사용하는 /dev/urandom은 요청하는 바이트만큼 즉시 데이터를 가져온다. /dev/urandom은 한번 암호학적으로 생성된 유사난수를 seed로 사용 status를 갱신함으로써 새로운 entropy 없이 무한히 난수를 생성한다.

이렇게 보면 사실 /dev/random에 데이터가 더 true random에 가깝다는 걸 알 수 있다. 하지만 만약 한 Application에서 entropy pool에 데이터를 독점하는 **DOS** 취약점이 발생하면 /dev/random 보다는 암호학적으로 충분히 안전한 /dev/urandom을 사용하는게 더 효율적이다.

## 추가
1. 사실 생각보다 리눅스에 Entropy Pool 생성은 더 복잡하다. Mouse, Keyboard, Interrupts, IDE 심지어 Microphone, Intel CPU 등 매우 복잡하고 수학적인 방법으로 Entropy pool를 생성한다.  
![DRNG](http://images.bit-tech.net/content_images/2011/10/all-about-ivy-bridge/drng-w.jpg)
  
2. 시스템 재부팅시 entropy pool이 초기화되는 현상을 막기 위해서 /dev/urandom은 종료시 데이터를 파일로 저장한 후 부팅 시 다시 불러온다.  


## 참고

https://en.wikipedia.org/wiki/Pseudorandom_number_generator  
https://eprint.iacr.org/2012/251.pdf  
https://elixir.bootlin.com/linux/v4.3/source/drivers/char/random.c  
