# 운영 체제
운영 체제는 프로그래머들의 작업의 중심에 자리하고 있습니다. (운영체제나 하드웨어를 만드는 사람들 조차도요)
그렇기 때문에 운영체제를 빼놓고는 어떠한 프로그래밍의 법칙도 토론할 수 없습니다.

The operating system stands in the centre of everything we do as programmers (well, unless you're [writing an Operating System](https://os.phil-opp.com/) or are in the [Embedded realm](https://rust-embedded.github.io/book/)),
so there is no way for us to discuss any kind of fundamentals in programming
without talking about operating systems in a bit of detail.

## 운영체제의 관점에서 바라본 동시성
## Concurrency from the operating systems' perspective

<div style="color: back;  font-style: italic; font-size: 1.2em">"Operating systems has been "faking" synchronous execution since the 90s."</div>
운영체제는 90년대 부터 마치 동기적 실행을 하고 있는 것처럼 동작했었습니다.

이는 챕터 1에서 동시적인 이라는 단어를 설명했을 때와 연결됩니다.
동시적이라는 단어는 OS가 프로세스를 언제든지 멈추거나 재개할 수 있다는 전제조건하에 이야기 할 필요가 있습니다.

동기적 코드라는 것은 보통 프로그래머의 관점에서 바라본 것입니다. OS나 CPU 모두 완전히 동기적으로 작동하지는 않는데도 말입니다.
This ties into what I talked about in the first chapter when I said that `concurrent`
needs to be talked about within a reference frame and I explained that the OS
might stop and start your process at any time.

What we call synchronous code is in most cases code that appears as synchronous to us as programmers. Neither the OS or the CPU live in a fully synchronous world.

운영체제는 `선점형 멀티태스킹`을 사용합니다.
만약 사용하고 있는 운영체제가 선점적으로 스케쥴링하는 한 소스 코드가 중단 없이 한줄 한줄 실행된다고 보장할 수 없습니다.

운영체제는 중요한 프로세스들을 실행시키기위해 CPU로 부터 일정 시간을 할당받을 것 입니다.

> 여러개의 물리적 코어를 가지고 있는 현대 컴퓨터들에 대해 이야기 하고 있다면 
> 그 이유는 
> 여기서 가장 중요한 부분은 
> This is not as simple when we're talking about modern machines with 4-6-8-12
> physical cores since you might actually execute code on one of the CPU's
> uninterrupted if the system is under very little load. The important part here
> is that you can't know for sure and there is no guarantee that your code will be
> left to run uninterrupted.


## 운영체제와 함께 일하기

프로그래밍을 하다보면 효율성에 대해 신경쓸 때 얼마나 많은 부품들이 협업해야 하는지 잊기 쉽습니다.
웹에서 요청을 만들 때 CPU나 네트워크 카드에 작업을 요청하지는 않습니다. 운영체제에게 네트워크 카드를 사용해달라고 요청할 뿐입니다.

운영체제의 장점을 살리지 않고서 프로그래머가 본인의 시스템을 가장 효과적으로 만들 수 있는 방법은 없습니다.
프로그래머들은 기본적으로 하드웨어에 직접적으로 접근할 필요가 없습니다. 

다른말로 이야기하면, 이 모든것들을 기초부터 탄탄히 이해하기 위해서는 운영체제가 작업들을 어떻게 다루는지 알아야할 필요가 있다는 말이 됩니다.
운영체제와 함께 일하기 위해 프로그래머들이 어떻게 운영체제와 통신하는지에 대해 알아야합니다. 바로 다음에서 이 내용을 살펴보도록 하겠습니다.
