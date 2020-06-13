# 운영체제와 소통하기

**이번 챕터에서는 다음 내용들에 대해 자세히 알아보겠습니다.:**

- 시스템콜이 무엇인지
- 추상화 레벨
- 저수준 크로스 플랫폼 코드를 작성하는 도전

## Syscall 초급
운영체제와의 소통은 우리가 지금부터 "syscall" 이라고 부를 `시스템 호출`을 통해 이루어 집니다.
이것은 운영체제가 제공하는 전체 공개 API로, 사용자 환경에서 작성 된 프로그램이 운영체제와 소통하기 위해 사용할 수 있습니다.

대부분 이 시스템 콜은 언어 단이나 런타임 단에서 추상화 되어있습니다.
Rust 같은 언어는 아래에 살펴볼 `syscall`을 만들기 쉽게 해줍니다.

`syscall`은 현재 소통하고 있는 커널에 대해 단 하나만 존재하는 것에 대한 표본입니다,
그러나 UNIX 패밀리의 커널들은 많은 공통점이 존재합니다. UNIX 시스템들은 **`libc`** 를 통해 시스템콜을 노출시킵니다.

반면에 Windows는 Windows 자체의 API를 사용하고, 주로 WinAPI 라고 불리며, UNIX 기반의 시스템이 작동되는 방식과 근본적으로 다릅니다.

대부분의 경우, 다른 운영체제더라도 같은 작업을 성취할 수 있는 방법이 있습니다. 
기능 측면에서는 큰 차이점이 없다고 느낄 수 있겠지만 아래에 다루고 있는 내용처럼  `epoll`, `kqueue`, `IOCP`에 대해 깊게 알게 될 수록 각 기능들이 구현된 방법이 매우 다르다는 것을 알 수 있습니다.

## Syscall 예제

`syscall`과 조금 더 친해지기 위해, `BSD(macos)`, `Linux`, `Windows` 이 세가지의 설계를 위한 기초를 구현해보겠습니다.
이 단계에서 3가지 단계의 추상화가 어떤식으로 구현되는지 확인할 수 있습니다.

지금부터 `표준 출력`을 실행시킬 때 사용되는 `syscall`을 구현해보겠습니다. 이것은 흔한 작동이며 매우 흥미롭습니다.


### 추상화의 가장 낮은 단계

앞서 말한 것들을 실현하기 위해서는 [인라인 어셈블리](https://doc.rust-lang.org/1.0.0/book/inline-assembly.html)를 작성해야 합니다.
CPU에 작성할 명령들 부터 알아보겠습니다.

> 만약 인라인 어셈블리에 대한 더많은 정보를 알고 싶다면, [글쓴이가 전에 작성해둔 gitbook](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/an-example-we-can-build-upon)을 추천합니다.

이 추상화 레벨에서는, 세가지 플랫폼별로 다른 코드를 작성하게 됩니다.
Linux와 Macos 에서는 실행하려는 `syscall`을 `write`라고 부릅니다. 두 시스템 모두 `파일 기술자`라는 개념에 기반해 동작하고, `표준출력`은 프로세스를 구동할 때 이미 존재하는 시스템 콜 중 하나입니다.

**리눅스에서 `write` 시스템콜은 이런식으로 작성됩니다 \
(우측 상단의 화살표 버튼을 눌러 코드를 실행해볼 수 있습니다)
```rust
#![feature(llvm_asm)]
fn main() {
    let message = String::from("Hello world from interrupt!\n");
    syscall(message);
}

#[cfg(target_os = "linux")]
#[inline(never)]
fn syscall(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    
    unsafe {
        llvm_asm!("
        mov     $$1, %rax   # system call 1 is write on linux
        mov     $$1, %rdi   # file handle 1 is stdout
        mov     $0, %rsi    # 출력할 문자열의 주소
        mov     $1, %rdx    # 바이트의 갯수
        syscall             # 커널을 호출, syscall interrupt 발생
    "
        :
        : "r"(msg_ptr), "r"(len)
        : "rax", "rdi", "rsi", "rdx"
        )
    }
}
```

리눅스에서 `write` 시스템콜을 개시하는 코드는 `1`이기 때문에 `$$1`이라고 작성하면  `1`이라는 값(literal value)이 그대로 `rax` 레지스터에 쓰이게 됩니다.

> `$$`은 AT&T 문법을 사용하는 인라인 어셈블리에서 값(literal value)을 작성하는 방법입니다. `$`는 파라미터로 참조하고 있는 것을 뜻합니다. `$0` 이라고 작성한다면 `msg_ptr`이 실행될 때 첫번째 인자를 참조합니다.

> (코드의 가장 하단 부분) 사용한 레지스터를 clobber 처리 해야 합니다. 이는 컴파일러에게 특정 레지스터들을 수정하고 있다는 것과 
그 안에 어떤 값을 저장하든 신뢰할 수 없다는 것을 알려줍니다.

첫번째로, `1`이라는 값을 `rdi` 레지스터에 넣는것은 쓰기 작업을 실행하고자 하는 파일 기술자인 `표준 출력`을 참조한다는 것을 뜻합니다.
`write` 시스템 콜이 `1`이라는 코드를 사용한다는 것과는 아무런 관계가 없습니다.

두번째로 문자열 버퍼의 주소와 버퍼의 길이를 각각 `rsi`와 `rdx` 레지스터에 넘긴 후, `syscall` 명령을 호출합니다.

> `syscall` 명령은 비교적 최근에 등장했습니다. `x86` 아키텍처에서 32비트 시스템의 초기에는, `int 0x80`이라는 소프트웨어 인터럽트를 발생시킴으로써 syscall을 호출했습니다. 소프트웨어 인터럽트는 지금 살펴보고 있는 저수준 단계에서 느린것으로 간주되어 후에 `syscall`이라는 이름의 분리된 명령이 추가되었습니다.

`syscall` 명령은 [VDSO](http://articles.manugarg.com/systemcallinlinux2_6.html)를 사용합니다. 이 VDSO는 각 프로세스의 메모리에 메모리 페이지를 매핑해주는 것 이기 때문에, 시스템 콜을 실행하는데 있어서 문맥교환이 필요하지 않습니다.

The `syscall` instruction uses [VDSO](http://articles.manugarg.com/systemcallinlinux2_6.html), which is a memory page attached to each process' memory, so no context switch is necessary to execute the system call.

**MacOS에서는, `syscall`이 이런 형태로 보여집니다:** \
(Rust playgrond는 리눅스 기반으로 작동하기 때문에, 이 예제는 실행할 수 없습니다)

```rust, 작동 안됨
#![feature(llvm_asm)]
fn main() {
    let message = String::from("Hello world from interrupt!\n");
    syscall(message);
}

#[cfg(target_os = "macos")]
fn syscall(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    unsafe {
        llvm_asm!(
            "
        mov     $$0x2000004, %rax   # system call 0x2000004 is write on macos
        mov     $$1, %rdi           # file handle 1 is stdout
        mov     $0, %rsi            # address of string to output
        mov     $1, %rdx            # number of bytes
        syscall                     # call kernel, syscall interrupt
    "
        :
        : "r"(msg_ptr), "r"(len)
        : "rax", "rdi", "rsi", "rdx"
        )
    };
}
```

`write`라는 syscall이 `1`대신 `0x2000004`라는 코드를 사용한다는 점 외에는 Linux 코드와 다른점이 없습니다.

**Windows는 어떨까요?**

지금까지 작성해온 코드가 좋지 않은 생각이라는 것을 설명할 좋을 기회인 것 같습니다.

코드가 오랜시간동안 정상적으로 작동하려면 OS가 제공하는 것들을 무엇이 `보장해주는지` 걱정할 필요가 있습니다. 
Linux와 MacOS는 어느정도 수준을 보장하고 있습니다. 예를 들어 `$$0x2000004`는 MacOS에서 항상 `write`를 뜻하고 있습니다. (이 보증이 얼마나 강력한지는 확신할 수 없습니다만)
Windows는 저수준의 내부동작에 관해서는 전혀 보증하지 않는다고 단언할 수 있습니다.

Windows는 내부동작을 셀수 없는 만큼 많이 바꿔왔고 그에 대해 공식 문서를 제공하지도 않았습니다. [이러한](https://j00ru.vexillium.org/syscalls/nt/64/) 리버스 엔지니어링 결과만을 알 수 있을 뿐입니다.
이것은 `write` 였던 명령어가 다음 Windows 업데이트 때에는 `delete`로 변경될 수 있다는 뜻입니다.

## 추상화의 다음 단계

추상화의 다음 단계는 이 세가지 운영체제들이 제공하는 API를 사용하는 것 입니다.

이 추상화가 불필요한 코드를 삭제하는데 도움을 준다는 사실은 이미 확인했습니다. syscall은 Linux와 MacOS에서는 동일하기 때문에 Windows 환경에만 신경을 쓰면 됩니다. 따라서 `#[cfg(not(target_os = "windows"))]` 라는 조건부 컴파일 flag를 사용하게 됩니다. Window환경에서는 반대로 진행할 것입니다.

### Linux와 MacOS에서 제공하는 API 사용하기

이 코드를 바로 실행해 볼 수 있습니다.
그러나 Rust playground는 Linux에서 동작하기 때문에, 아래 코드를 Windows 에서 확인해 보기 위해서는 Windows 머신에 복사해야 합니다.

**syscall이 이렇게 보이게 됩니다.**  \
(Linux와 MacOS에서 동작하는 코드입니다.)

```rust
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

// and: http://man7.org/linux/man-pages/man2/write.2.html
#[cfg(not(target_os = "windows"))]
#[link(name = "c")]
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize) -> i32;
}

#[cfg(not(target_os = "windows"))]
fn syscall(message: String) -> io::Result<()> {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    let res = unsafe { write(1, msg_ptr, len) };

    if res == -1 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

```

지금부터 코드에 대해 main 부분은 생략하고 설명하겠습니다.


```rust, no_run
#[link(name = "c")]
```

모든 Linux 버전은 `libc` 버전을 가지고 있습니다. `libc`는 운영체제와 소통하기 위한 C언어 라이브러리 입니다. 
일관된 API와 `libc`를 사용한다는 것은 다른 코드에 영향을 주지 않으면서 가장 밑단의 구현체들을 변경할 수 있다는 뜻입니다.
이 구문은 컴파일러가 현재 컴파일이 일어나고 있는 시스템에 C라이브러리를 연결하도록 만듭니다.

```rust, no_run
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize);
}
```


`extern "C"` or only `extern` (C is assumed if nothing is specified) means we're linking to specific functions in the "c" library using the "C" calling convention. As you'll see on Windows we'll need to change this since it uses a different calling convention than the UNIX family.

The function we're linking to needs to have the exact same name, in this case, `write`. The parameters don't need to have the same name but they must be in
the right order and it's good practice to name them the same as in the library
you're linking to.

The write function takes a `file descriptor` which in this case is a handle to
`stdout`. In addition, it expects us to provide a pointer to an array of `u8` values and the length of the buffer.

```rust, no_run
#[cfg(not(target_os = "windows"))]
fn syscall_libc(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    unsafe { write(1, msg_ptr, len) };
}
```
First, we get a pointer to the underlying buffer of our
string. This will be a pointer of type `*const u8` which matches our `buf`
argument. The `len` of the buffer corresponds to the `count` argument.

You might ask how we know that `1` is the file-handle to `stdout` and where we found that value.

You'll notice this a lot when writing syscalls from Rust. Usually, constants are defined in the C header files which we can't link to, so we need to search them up. 1 is always the file descriptor for `stdout` on UNIX systems.

> Wrapping the `libc` functions and providing these constants is exactly what the
> crate [libc](https://github.com/rust-lang/libc) provides for us and why you'll see that used instead of writing
> the type of code we do here.

A call to an FFI function is always unsafe so we need to use the `unsafe` keyword
here.


### Using the API on Windows

**This syscall will look like this on Windows:** \
(You'll need to copy this code over to a Windows machine to try this out)
```rust, no_run
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}

#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe {
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
            };
        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output as usize, len);
    Ok(())
}
```

Now, just by looking at the code above you see it starts to get a bit more
complex, but let's spend some time to go through line by line what we do here as
well.

```text
#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
```

The first line is just telling the compiler to only compile this if the `target_os` is Windows.

The second line is a linker directive, telling the linker we want to link to the library `kernel32` (if you ever see an example that links to `user32` that will also work).


```rust, no_run
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}
```

First of all, `extern "stdcall"`, tells the compiler that we won't use the `C`
calling convention but use Windows calling convention called `stdcall`.

The next part is the functions we want to link to. On Windows, we need to link to two functions to get this to work: `GetStdHandle` and `WriteConsoleW`.
`GetStdHandle` retrieves a reference to a standard device like `stdout`.

`WriteConsole` comes in two flavours, `WriteConsoleW` that takes in Unicode text and `WriteConsoleA` that takes ANSI encoded text.

Now, ANSI encoded text works fine if you only write English text, but as soon as you write text in other languages you might need to use special characters that are not possible to represent in `ANSI` but is possible in `utf-8` and our program will break.

That's why we'll convert our `utf-8` encoded text to `utf-16` encoded Unicode codepoints that can represent these characters and use the `WriteConsoleW` function.

```rust, no_run, noplaypen
#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe {
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
            };

        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output, len);
    Ok(())
}
```
The first thing we do is to convert the text to utf-16 encoded text which
Windows uses. Fortunately, Rust has a built-in function to convert our `utf-8` encoded text to `utf-16` code points. `encode_utf16` returns an iterator over  `u16` code points that we can collect to a `Vec`.

```rust, no_run, noplaypen
let msg: Vec<u16> = message.encode_utf16().collect();
let msg_ptr = msg.as_ptr();
let len = msg.len() as u32;
```

Next, we get the pointer to the underlying buffer of our `Vec` and get the
length.

```rust, no_run, noplaypen
let handle = unsafe { GetStdHandle(-11) };
   if handle  == -1 {
       return Err(io::Error::last_os_error())
   }
```

The next is a call to `GetStdHandle`. We pass in the value `-11`. The values we
need to pass in for the different standard devices is actually documented
together with the `GetStdHandle` documentation:

|Handle|Value|
|------|-----|
|Stdin|-10|
|Stdout|-11|
|StdErr|-12|


Now we're lucky here, it's not that common that we find this information
together with the documentation for the function we call, but it's very convenient when we do.

The return codes to expect is also documented thoroughly for all functions so we handle potential errors here in the same way as we did for the Linux/MacOS syscalls.

```rust, no_run
let res = unsafe {
    WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
    };

if res  == 0 {
    return Err(io::Error::last_os_error());
}
```

Next up is the call to the `WriteConsoleW` function. There is nothing too fancy about this.

## The highest level of abstraction

This is simple, most standard libraries provide this abstraction for you. In rust that would simply be:

```rust
println!("Hello world from Stdlib");
```


# Our finished cross-platform syscall

```rust
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

// and: http://man7.org/linux/man-pages/man2/write.2.html
#[cfg(not(target_os = "windows"))]
#[link(name = "c")]
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize) -> i32;
}

#[cfg(not(target_os = "windows"))]
fn syscall(message: String) -> io::Result<()> {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    let res = unsafe { write(1, msg_ptr, len) };

    if res == -1 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}

#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe {
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
            };

        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output, len);
    Ok(())
}
```
