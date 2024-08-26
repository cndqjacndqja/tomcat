NIO, IO 내부 코드가 어떻게 동작했고 발전해왔는지 알아보자.

JIOEndpoint는 7.0 버전까지 사용되다가 그 이후부터 Deprecated 되었다. 그 이유는 NIO를 사용하는 방향으로 바뀌었기 때문이다.
Spring boot는 1.0 버전부터 NIO를 지원하고 있었고, 2.0부터 NIO가 기본 설정이 되었다.

### IO 처리 과정의 한계와 문제점은?
처음 내 생각: Acceptor 스레드가 블로킹되어 있어서, 새로운 연결을 받지 못하는 문제가 발생할 수 있다.
-> 그러나 아니라고 한다. Acceptor 스레드는 새로운 연결을 받는 역할만 하고 바로 ThreadPool에 토스해버리기 때문에, 이 부분에서 병목이 일어날 경우는 매우 드물다고 한다.

실제 문제가 되는 부분: ThreadPool에서 처리하는 부분에서 발생한다.
- ThreadPool의 동작
    - ThreadPool은 미리 정의된 수의 스레드를 생성합니다(예: 최소 10개, 최대 200개).
    - 모든 스레드가 사용 중일 때 새로운 연결이 들어오면:
    - a) 새 스레드를 생성합니다(최대 한도까지).
    - b) 또는 요청을 큐에 넣고 대기시킵니다.
- 여기서 문제점
    - 동시 연결 수가 ThreadPool의 최대 스레드 수를 초과하면:
      a) 새 연결은 큐에서 대기하게 되어 응답 시간이 길어집니다.
      b) 메모리 사용량이 증가합니다(각 스레드는 메모리를 소비).
- 구체적인 시나리오:
    - ThreadPool 최대 크기: 200
    - 동시 연결 수: 1000
    - 결과: 200개의 연결은 즉시 처리됩니다. 나머지 800개는 큐에서 대기하거나 연결이 거부될 수 있습니다.

요약: 결국 IO 방식은 스레드 풀을 만들어 놓고, 요청당 하나의 스레드를 사용하는 것인데
예를 들어 스레드풀에 200개 넣어놓으면, 요청이 올 때 마다 이 중 하나를 사용할텐데 요청이 한번에 300개가 오면 한번에 고갈이 되어버리기 때문에 한계가 있었다.
그리고 요청당 하나의 스레드를 사용ㅎ는건 스레드 간의 컨텍스트 스위칭으로 인한 오버헤드가 발생하여 성능 저하를 일으킬 수 있다고 한다.

### 찐 내부 구현체는 어디있을까
- IO와 NIO의 socket요청을 받는 부분부터 다르다.
  - IO는 ServerSocket을 사용한다. (가장 밑단은 native인 c로 구현되어있다.)
    - serverSocket.accept()은 요청이 올 때까지 블로킹을 한다.
  - NIO는 ServerSocketChannel을 사용한다.
    - channel을 selector에 등록한다.
    - channel 객체를 여러개 만들어서 selector에 등록하니, serverSocket.accept()처럼 블로킹되지 않는다. 왜냐면 serverSocket은
    - 요청 listen하는 하나의 스레드를 쭉 점유하고 있는 것이고, selector는 channel이라는 "객체(스레드가 아닌)"를 여러개 만들어서 등록 후 요청을 listen하는 것이기 때문이다.

### 궁금증(처음엔 이해가 안됐던 부분)
- 그럼 selector도 요청을 받는 쪽만 객체로 받아서 결국 스레드 풀로 위임하는건데, 그럼 스레드풀 200개 설정이고, 요청 한번에 200개 넘게 오면 IO랑 동일한 문제가 있는 것은 아닌가?
> 만약 300개의 요청이 왔다고 가정하자. IO는 200개의 요청을 먼저 처리하고 100개는 아예 대기이거나 요청 자체를 받지 못한다.
> 하지만 NIO는 channel이라는 객체로 일단 300개 요청을 다 받아놓고, 스레드 풀로 위임하여 200개의 스레드풀로 동시에 처리하고 100개는 큐에 넣어놓는다.
> 그럼 결국 IO(ex. JIoEndpoint)랑 NIO(ex. NioEndpoint)의 차이점은 요청을 수신하는 쪽에 있다. IO는 요청이 오면 스레드풀로 위임하기 전까지 요청 수신을 못하는 반면, NIO는 이부분은 channel이라는 객체와 selector라는 객체를 사용함으로써
> 하나의 스레드에서 여러개의 수신해서 스레드 풀로 위임할 수 있다는 차이가 핵심이다.

### 그럼 여기서 스프링과 연관지어 생각해보자.
server.tomcat.max-connections=10000
server.tomcat.accept-count=100
server.tomcat.threads.max=200
기존에 위와 같은 설정값들을 대충 이해하고 설정했었다.
그럼 이제 이 설정값들이 어떤 의미인지 알 수 있을 것이다.

- max-connections: 최대 연결 수인데, 이게 코드단에서는 channel객체의 개수이다. 요청올 떄 딱 수신하는 그 부분의 객체 개수를 지정하는 것이고, 이 개수만큼 한번에 요청을 listen할 수 있다.
- accept-count는 큐의 개수이다. channel의 개수보다 한번에 더 많은 요청이 올 경우 이 큐에 넣어놓는데, 이것의 개수를 지정하는 것이다.
- threads.max는 스레드 풀의 개수이다.

> 그럼 정리를 하자면,, max-connections로 지정된 channel의 개수만큼 소켓 요청을 listen할 수 있는 것이고, 이것보다 한번에 많은 요청 들어오면 accept-count만큼의 사이즈인 큐에 넣어놓는 것이고, 이렇게 요청을 listen했으면 스레드풀로 위임하는데 그 스레드풀 사이즈를 지정하는 것이 threads.max이다!

### NIOEndpoint에 대해 알아보자.
- Poller
    - Poller가 실제로 Selector를 가지고 있으며, 이를 통해 I/O 이벤트를 관리한다.
    - NIOEndpoint는 하나 이상의 Poller를 가질 수 있으며, 각 Poller는 하나의 Selector를 가진다.
    - NIO는 일반적으로 여러 개의 Poller를 사용한다. 이는 멀티 코어 시스템에서 성능을 향상시키기 위함이다.
- Accepter
- Selector
    - Poller 내부의 run() 메서드에서 Selector를 사용해서 I/O 이벤트를 처리한다.

> 기본적으로 tomcat에서 poller의 개수는 2개이고, 1000개의 max-connections를 설정했다면 1000개의 channel을 2개의 poller가 나눠서 처리한다는 것이다.

### NioEndpoint에서 Poller가 이벤트 수신하는 딱 그 부분은 어디에 있을까?
```java

PollSelectorImpl.class -> doSelect(Consumer<SelectionKey> action, long timeout) 메서드


do {
    long startTime = timedPoll ? System.nanoTime() : 0;
    numPolled = poll(pollArray.address(), pollArraySize, to);
    if (numPolled == IOStatus.INTERRUPTED && timedPoll) {
    // timed poll interrupted so need to adjust timeout
    long adjust = System.nanoTime() - startTime;
    to -= TimeUnit.MILLISECONDS.convert(adjust, TimeUnit.NANOSECONDS);
    if (to <= 0) {
    // timeout expired so no retry
    numPolled = 0;
    }
  }
} while (numPolled == IOStatus.INTERRUPTED);

```

위 부분에서 계쏙 while문 돌면서 반환하지 않고 있다가, INTERRUPTED 상태가 되어 이벤트를 수신했다는 것을 감지하면 return한다.
(왜 근데 이벤트 루프라던지 이벤트 수신한다는 표현을 굳이 어렵게 쓰는걸까? 결국 내부 구현은 while문 돌다가 반환하거나 재귀도는 것들이 대부분인데, 이러한 표현들이 괜히 더 어렵게 만드는 것 같다,,..)

-> 아.. 좀 더 확인해보니  INTERRUPTED 상태가 될때까지 while문을 도는 것이 아니라, poll()메서드 호출 시 블로킹 되고 요청올때까지 대기하는거구나. poll메서드는 native 메서드로 구현되어있고, 운영 체제의 I/O 다중화 시스템 콜을 사용한다고 한다.
INTERRUPTED 상태도 뜻을 잘못 이해했었다.
즉, 이벤트 루프라던지 이벤트 수신이 항상 내부 구현이 while문으로 되어있는 것은 아니군,,

### poll과 같은 native 메서드는 이벤트 대기를 어떻게 할까?
대부분의 application 단에서 이벤트를 캐치하는 부분은 while, 재귀를 도는 것으로 봤었다.
근데 NIO같이 소켓을 통해 HTTP요청을 받아 처리하는 이 부분도 과연 while, 재귀를 사용할까? (정말 이걸로만 구현해도 문제가 없는가에 대한 의문이 들었다.)
-> native 메서드인 poll은 내부 구현은 epoll_wait함수를 사용하고 이 함수는 커널 공간에서 동작한다.
이것의 이벤트 대기 매커니즘은 다음과 같다.
- 커널은 내부적으로 이벤트 테이블이나 큐를 관리한다.
- epoll_wait호출 시, 프로세스/스레드는 대기(WAIT)  상태로 전환되고 이 상태에서는 CPU를 사용하지 않는다. (얼핏 OS 수업에서 들었던 내용이 기억난다.)
  - 이 부분 관련해서는 좀 더 깊게 공부하고 싶지만, 일단은 이정도로만 이해하고 넘어가자. 암튼 찐 내부는 native 메서드! 커널 이용해서 이벤트 대기하고 이때는 CPU 안씀!






