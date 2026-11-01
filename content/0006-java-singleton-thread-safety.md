---
title: "싱글턴은 스레드 안전성을 보장할까?"
preview: "싱글턴은 인스턴스를 하나로 유지하고 여러 곳에서 공유 할 수 있는 디자인 패턴 입니다. 하지만 싱글턴인 인스턴스 메서드를 동시에 실행하면 어떤 변화가 있을까요?"
tags: [java, concurrency, singleton, thread-safety]
---

싱글턴(Singleton)은 인스턴스를 하나만 생성하고 여러 곳에서 공유하도록 하는 디자인 패턴입니다. 하지만 인스턴스가 하나라는 사실이 멀티스레드 환경에서의 안전성까지 보장하지는 않습니다.

**예제)**

```java
public final class Counter {
    private static final Counter INSTANCE = new Counter();
    private int value = 1;

    private Counter() {}

    public static Counter getInstance() {
        return INSTANCE;
    }

    public int increment() {
        return ++value;
    }
}
```

위 코드는 `static final` 필드로 싱글턴 인스턴스를 생성하므로 클래스 초기화 과정에서 스레드 안전하게 초기화됩니다. 하지만 여러 스레드가 `increment()`를 동시에 호출하면 문제가 발생할 수 있습니다.

Why?

`++value`는 값을 Lead, Write, save하는 복합 연산입니다.

1. 스레드 A가 `value`의 값 `1`을 읽습니다.
2. 스레드 B도 `value`의 값 `1`을 읽습니다.
3. 스레드 A가 `2`를 저장합니다.
4. 스레드 B도 `2`를 저장합니다.

두 번 증가했지만 최종 값은 `2`가 될 수 있습니다. 이를 경쟁 상태(Race Condition)라고 합니다.

Java에서는 동기화되지 않은 공유 변수의 동시 접근으로 데이터 레이스가 발생할 수 있습니다. 단순히 증가 연산이 누락되는 것뿐 아니라, 스레드 간 변경 사항이 올바르게 관찰되지 않는 문제도 발생할 수 있습니다.

**원자성(Atomicity)이란?**

원자성은 연산이 중간 단계로 나뉘어 다른 스레드와 충돌하지 않고, 하나의 연산처럼 수행되는 성질입니다.

일반 정수 변수의 `++value`는 원자적이지 않습니다. 이를 해결하는 대표적인 방법은 `AtomicInteger`, `synchronized`, `ReentrantLock`입니다.

1. AtomicInteger 사용

```java
import java.util.concurrent.atomic.AtomicInteger;

public final class Counter {
    private static final Counter INSTANCE = new Counter();
    private final AtomicInteger value = new AtomicInteger(1);

    private Counter() {}

    public static Counter getInstance() {
        return INSTANCE;
    }

    public int increment() {
        return value.incrementAndGet();
    }
}
```

`incrementAndGet()`은 증가 연산을 원자적으로 수행합니다. 두 스레드가 동시에 호출하면 각각 `2`, `3`을 반환하고 최종 값은 `3`이 됩니다.

단, `AtomicInteger`는 모든 코드의 원자성을 보장하는 것이 아니라, 제공하는 개별 연산의 원자성을 보장합니다.

2. synchronized 사용

```java
public synchronized int increment() {
    return ++value;
}
```

`synchronized`는 같은 모니터를 획득한 스레드끼리 동시에 진입하지 못하도록 합니다.
한 스레드가 해당 메서드를 실행하는 동안 다른 스레드는 같은 모니터를 획득할 때까지 대기합니다.

메서드 전체에 적용할 수도 있고, 메서드 내부의 특정 구간에만 적용할 수도 있습니다.
보호 범위가 넓을수록 불필요한 대기가 늘어날 수 있으므로 필요한 구간을 적절히 설정해야 합니다.

3. ReentrantLock 사용

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

private final Lock lock = new ReentrantLock();

public int increment() {
    lock.lock();
    try {
        return ++value;
    } finally {
        lock.unlock();
    }
}
```

`ReentrantLock`은 명시적으로 락을 획득하고 해제하는 방식입니다. `tryLock()`으로 락 획득을 즉시 시도하거나, `lockInterruptibly()`로 대기 중 인터럽트에 대응하는 등 `synchronized`보다 세밀한 제어 기능을 제공합니다.

`unlock()`을 직접 호출해야 하므로 `finally`에서 해제하는 것이 중요합니다.

**AtomicInteger와 Lock은 언제 사용할까?**

- 단순한 카운터의 증가·감소: `AtomicInteger`
- 여러 작업을 하나의 임계 영역으로 보호: `synchronized` 또는 `ReentrantLock`
- 락 획득 시간 제한, 인터럽트 대응, 공정성 설정 등이 필요한 경우: `ReentrantLock`

`AtomicInteger`는 개별 연산의 원자성을 제공하고, 락은 여러 단계의 작업을 임계 영역으로 묶어 동시 접근을 제어할 수 있습니다. (논블로킹)
