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
PRNG's seed 즉 예전 우리가 random 함수를 사용할 때 현재 시각으로 설정하곤 했던 그 seed 값과 내부 난수 발생 알고리즘을 통해 만들어진게 바로 유사난수다. 흔하게 유닉스 계열 운영체제에서 사용하는 /dev/random, /dev/urandom 장치 파일들이 바로 유사난수 발생기 역할을 한다.    


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

### blocking, nonblocking pool

아래 소스를 보면 random_read 함수에서는 blocking_pool, urandom_read 함수에서는 nonblocking_pool을 사용한다는 걸 확인할 수 있다.
```c
static ssize_t
_random_read(int nonblock, char __user *buf, size_t nbytes)
{
	ssize_t n;

	if (nbytes == 0)
		return 0;

	nbytes = min_t(size_t, nbytes, SEC_XFER_SIZE);
	while (1) {
		n = extract_entropy_user(&blocking_pool, buf, nbytes);
		if (n < 0)
			return n;
		trace_random_read(n*8, (nbytes-n)*8,
				  ENTROPY_BITS(&blocking_pool),
				  ENTROPY_BITS(&input_pool));
		if (n > 0)
			return n;

		/* Pool is (near) empty.  Maybe wait and retry. */
		if (nonblock)
			return -EAGAIN;

		wait_event_interruptible(random_read_wait,
			ENTROPY_BITS(&input_pool) >=
			random_read_wakeup_bits);
		if (signal_pending(current))
			return -ERESTARTSYS;
	}
}

static ssize_t
urandom_read(struct file *file, char __user *buf, size_t nbytes, loff_t *ppos)
{
	int ret;

	if (unlikely(nonblocking_pool.initialized == 0))
		printk_once(KERN_NOTICE "random: %s urandom read "
			    "with %d bits of entropy available\n",
			    current->comm, nonblocking_pool.entropy_total);

	nbytes = min_t(size_t, nbytes, INT_MAX >> (ENTROPY_SHIFT + 3));
	ret = extract_entropy_user(&nonblocking_pool, buf, nbytes);

	trace_urandom_read(8 * nbytes, ENTROPY_BITS(&nonblocking_pool),
			   ENTROPY_BITS(&input_pool));
	return ret;
}
```

## /dev/random vs /dev/urandom
그러면 도대체 두 장치 파일에 차이점은 무엇일까?  

blocking pool를 사용하는 /dev/random은 input_pool에서 추출한 데이터를 그대로 사용한다. 데이터를 가져올 시 entropy_count가 증가하고, 추출될 시 감소하는 방법으로 pool에 존재하는 데이터를 관리한다. 그래서 더 이상 추출될 데이터가 없으면 아래 소스같이 **entropy_count** 변수를 통해 확인한다.
```c
if (unlikely(entropy_count < 0)) {
  pr_warn("random: negative entropy count: pool %s count %d\n",
    r->name, entropy_count);
  WARN_ON(1);
  entropy_count = 0;
}
```

시스템상으로 nonblocking pool은 blocking pool보다 먼저 초기화된다. nonblocking pool을 사용하는 /dev/urandom은 요청하는 바이트만큼 데이터를 즉시 가져온다. blocking_pool과 다르게 새롭게 들어온 input pool에 데이터를 주기적으로 사용하는 게 아니라 한번 초기 설정된 nonblocking pool을 사용하여 유사난수를 발생시키고 이를 랜덤한 seed로 사용하여 주기적으로 PRNG의 status를 갱신한다. 이를 통해 지속적으로 **새로운** 유사난수를 생성할 수 있는 것이다.
```python
def isInitialized(self):
    # Read one byte from getrandom to determine whether the
    # nonblocking pool is initialized.
    try:
        r = self.getrandom(1, nonblock=True)
        if len(r) != 1:
            raise Exception("No data returned from getrandom")
        print("Nonblocking pool initialized")
        return True
    except GeneratorNotInitializedError:
        return False
```           

이렇게 보면 순수 input pool에 entropy만을 사용하는 /dev/random에서 추출한 난수가 더 안전하다는 걸 알 수 있다. 하지만 만약 한 Application에서 entropy pool에 데이터를 독점하는 **DOS** 취약점이 발생하거나, 단편화된 네트워크 장비에 들어가 있는 OS처럼 Entropy Sources가 마땅치 않을 경우 /dev/random은 실용적이지 못하다. 현재 기술로 urandom_read시 난수 발생기에서 새롭게 생성된 Status를 예측하기란 굉장히 어렵기 때문에 /dev/urandom을 사용하는 게 더 현실적이고 효율적이기 때문에 많은 OS, Application에서 이를 활용한다는 걸 알 수 있다.

## 추가
1. 사실 생각보다 리눅스에 Entropy Pool 생성은 더 복잡하다. Mouse, Keyboard, Interrupts, IDE 심지어 Microphone, Intel CPU 등 매우 복잡하고 수학적인 방법으로 Entropy pool를 생성한다.  
![DRNG](http://images.bit-tech.net/content_images/2011/10/all-about-ivy-bridge/drng-w.jpg)
  
2. 시스템 재부팅시 entropy pool이 초기화되는 현상을 막기 위해서 /dev/urandom은 종료시 데이터를 파일로 저장한 후 부팅 시 다시 불러온다.  


## 참고

https://en.wikipedia.org/wiki/Pseudorandom_number_generator  
https://eprint.iacr.org/2012/251.pdf  
https://elixir.bootlin.com/linux/v4.3/source/drivers/char/random.c  
https://github.com/openstack-infra/project-config/blob/master/nodepool/elements/initialize-urandom/static/usr/local/bin/initialize-urandom.py
