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

운영체제는 선점형 멀티태스킹을 사용합니다.
만약 사용하고 있는 운영체제가 선점적으로 스케쥴링한다면 프로그래머의 코드가 중단 없이 한줄 한줄 실행된다고 보장할 수 없습니다.
Operating systems use `preemptive multitasking` and as long as the operating system you're running is preemptively scheduling processes,
you won't have a
guarantee that your code runs instruction by instruction without interruption.

운영체제는 중요한 프로세스들은 
The operating system will make sure that all important processes get some time from the CPU to make progress.

> This is not as simple when we're talking about modern machines with 4-6-8-12
> physical cores since you might actually execute code on one of the CPU's
> uninterrupted if the system is under very little load. The important part here
> is that you can't know for sure and there is no guarantee that your code will be
> left to run uninterrupted.


## Teaming up with the OS.

프로그래밍을 할때 효율성에 신경쓰려면 어떤 부품들이 서로 상호작용해야하는지 잊기 쉽습니다.
웹에서 요청을 만들 때 CPU나 네트워크 보드에 작업을 요청하지는 않습니다. 운영체제에게 네트워크 보드를 사용해달라고 요청할 뿐입니다.
When programming it's often easy to forget how many moving pieces that need to
cooperate when we care about efficiency. When you make a web request, you're not
asking the CPU or the network card to do something for you, you're asking the
operating system to talk to the network card for you.


운영체제의 장점을 살리지 않고 프로그래머가 본인의 시스템을 매우 효과적으로 만들 수 있는 방법은 없습니다.
프로그래머들은 기본적으로 하드웨어에 직접적으로 접근할 필요가 없습니다. 
There is no way for you as a programmer to make your system optimally efficient
without playing to the strengths of the operating system. You basically don't have
access to the hardware directly.

그러나, 이것은 기초부터 탄탄히 이해하기 위해서, 운영체제가 작업들을 어떻게 관리하는지 알아야할 필요가 있습니다.
However, this also means that to understand everything from the ground up, you'll also need to know how your operating system handles these tasks.

To be able to work with the operating system, we'll need to know how we can communicate with it and that's exactly what we're going to go through next.

