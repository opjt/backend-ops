---
title: "Goroutine은 OS 스레드와 어떻게 다른가요?"
preview: "Go에서는 goroutine을 수십만 개씩 띄워도 거뜬합니다. 같은 일을 OS 스레드로 하기는 어려운데, 무엇이 이 차이를 만들까요?"
tags: [go, concurrency, runtime]
---

OS 스레드는 운영체제가 직접 관리하며, 생성 비용이 크고 스택 크기가 고정(보통 1~8MB)되어 있습니다. 컨텍스트 스위칭도 커널을 거치기 때문에 비용이 높습니다.

Goroutine은 Go 런타임이 관리하는 경량 실행 단위입니다. 초기 스택이 2KB에서 시작해 필요에 따라 동적으로 늘어나고, 컨텍스트 스위칭도 유저 스페이스에서 처리합니다. 덕분에 수십만 개를 동시에 띄워도 메모리와 CPU 부담이 크지 않습니다.

Go 런타임은 **M:N 스케줄링** 모델을 사용합니다. M개의 goroutine을 N개의 OS 스레드에 multiplexing하는 방식으로, 런타임 스케줄러(`GOMAXPROCS`로 제어)가 goroutine을 적절히 분배합니다.

```go
// OS 스레드 1개당 goroutine 수천 개가 실행될 수 있음
for i := 0; i < 100_000; i++ {
    go func() {
        // 각 goroutine은 ~2KB 스택으로 시작
    }()
}
```

goroutine이 I/O나 channel 대기 상태가 되면 런타임이 해당 OS 스레드에 다른 goroutine을 올려 CPU를 낭비하지 않습니다. 이것이 Go가 높은 동시성을 낮은 비용으로 처리할 수 있는 핵심 이유입니다.
