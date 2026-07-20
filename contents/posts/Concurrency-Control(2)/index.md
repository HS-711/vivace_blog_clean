---
title: "Concurrency-Control(2)"
description: 동시성 제어(2)
date: 2026-06-20
update: 2021-04-04
tags:
  - DB
  - Lock
  - Transaction
series: "DataBase"
---

이번에는 잠금 이외의 방법으로 동시성 제어를 하는 규약들에 대해 알아보자.
## Timestamp-Based
타임스탬프 기반 규약 기법을 사용하기 위해서 각 데이터에 두가지 타임스탬프를 둔다.
> - W-timestamp(Q): write(Q) 한 트랜잭션들 중에 가장 큰 값.<br/>
> - R-timestamp(Q): read(Q) 한 트랜잭션들 중에 가장 큰 값.<br/>

이때, T가 read(Q)를 수행하려고 한다고 해보자,
> - $TS(T)<=W-Timestmap(Q)$ 라면, T는 w가 덮어쓴 새로운 값을 읽어야 한다. 따라서 read(Q)는 거절되고 T1는 롤백된다.
> - $TS(T)>=W-Timestamp(Q)$ 라면, read(Q)는 정상적으로 실행되고, <br/>$R-Timestamp(Q)=max(R-Timestamp(Q),read(Q))$로 갱신된다.

또한, T가 write(Q)를 수행하려고 한다고 해보자,
> - $TS(T)<R-Timestamp(Q)$ 라면, Q는 나중에 읽으려고 했던 트랜잭션이 필요한 값으로, write(Q)를 거절하고 T1을 롤백한다.
> - $TS(T)<W-Timestamp(Q)$ 라면, 나중에 실행된 트랜잭션이 Q를 갱신했으므로,<br/> 시스템은 write(Q)를 거절하고 T1을 롤백한다.
> - 다른 경우는 write(Q)를 실행하고 $W-Timestamp(Q)=Write(Q)$로 갱신한다.

타임스탬프 기반 규약은 **충돌 직렬 가능성과 교착상태 해결**을 보장한다. <br/>
그러나 **회복가능성과 비연쇄성을 보장하지는 못한다.**

### Thomas-Write Rule
토마스 쓰기 규칙은 타임스탬프 규약을 개선한 방식이다.

> - $TS(T)<R-Timestamp(Q)$ 라면, Q는 나중에 읽으려고 했던 트랜잭션이 필요한 값으로, write(Q)를 거절하고 T1을 롤백한다.
> - $TS(T)<W-Timestamp(Q)$ 라면, 나중에 실행된 트랜잭션이 Q를 갱신했으므로,<br/> **시스템은 write(Q)를 거절한다.**
> - 다른 경우는 write(Q)를 실행하고 $W-Timestamp(Q)=Write(Q)$로 갱신한다.

타임스탬프 규약과 다른점은 **두번째 규칙**에 있다.<br/>
쓸모없는 write를 취소하여 transaction을 rollback하지 않고 무시한다.<br/>
단순히 명령어를 무시함으로서 **충돌 직렬가능성을 보장하지는 못한다. 그러나 뷰 직렬가능성을 제공한다.**

## Validation-Based

검증규약(Validation protocol)은 다음 단계에 거쳐 실행된다.
> 1. 읽기 단계(read phase): read(Q)는 값을 단지 해당 트랜잭션의 임시 지역변수에 저장한다.
> 2. 검증 단계(validation phase): 검증테스트를 통해 직렬가능성을 위배하지 않으면서 쓰기 단계로 진행하는 것을 허용할지 여부를 결정한다. 검증에 실패하면 해당 트랜잭션을 중단한다.
> 3. 쓰기 단계(write phase): 검증테스트에 성공하면 임시 지역변수에 있던 결과를 데이터베이스에 복사한다.
> 읽기 전용 트랜잭션은 이를 생략한다.

이를 위해 한 트랜잭션은 서로 다른 세개의 트랜잭션을 가진다.
> - Start(T): 실행을 시작한 시점
> - Validation(T): 읽기를 끝내고 검증 단계를 시작한 시점
> - Finish(T): 쓰기 단계를 끝낸 시점


검증테스트는 다음 중 하나를 만족해야한다.
> TS(T1) < TS(T2)인 트랜잭션에서
> 1. Finish(T1) < Start(T2)
> 2. Start(T2) < Finish(T1) < Validation(T2)이어야 한다.
> 아직 write되지 않은 데이터는 앞 트랜잭션에 영향을 주지 못하기 때문이다.

이 스케줄은 **직렬가능성은 보장하고 비연쇄성**을 보장한다.<br/>
커밋된 후에 데이터베이스에 실제 값을 기록하기 때문이다.

## Multiversion concurrency control(MVCC)
트랜잭션에서 write(Q)를 수행할 때 마다 Q에 대한 버전을 만든다. <br/> 
또한, read(Q)가 수행될 때, 어떤 버전의 Q를 읽을지 선택한다.<br/>
이때 선택하는 버전은 직렬가능성을 보장해야하고,<br/>
어떤 버전을 읽을지 대기없이 빠르게 결정할 수 있어야 한다.(no wait)<br/>

각 버전 $Q_K$는 세가지 데이터 필드를 가진다.
> $Cotent$: $Q_K$ 가 가지는 값<br/>
> $W-Timestamp(Q_K)$: $Q_K$ 을 생성한 타음스탬프<br/>
> $R-Timestamp(Q_K)$: $Q_K$ 을 읽은 가장 큰 타임스탬프

다중 버전 타임스탬프 순서기법은 직렬가능성을 보장하기 위해 다음과 같이 동작한다.

>트랜잭션 $T_J$이 다음과 같은 두가지 상황을 가질 수 있다. <br/>
> 1. $read(Q_K)$ 를 요청하면,<br/>
버전에 맞는 $Q_K$ 를 반환하고, $R-Timestamp(Q_K)$ 를 갱신한다.
> 2. $write(Q_K)$ 를 요청하면,<br/>
> - $TS(T_J) < R-Timestamp(Q_K)$ 라면, $T_J$ 은 취소되고 롤백된다.
> - $TS(T_J) = W-Timestamp(Q_K)$ 라면, $Q_K$ 는 덮어써진다.
> - 그렇지 않다면, 새로운 버전의 $Q_J$ 를 생성한다.<br/>

이러한 방식은 연산의 실패나 wait를 방지하지만, <br/>
read시 R-timestamp의 갱신으로 디스크 IO접근이 두번 필요하다는 것이다.
또한, 이 역시 직렬가능성을 보장하지만, 복구성과 비연쇄성을 보장하지 못한다.

## Snapshot Isolation
데이터 스냅샷이란 다음과 같은 특징을 가진다.
> - 시작 시점 고정: 트랜잭션 $T$가 시작될 때, 데이터베이스는 그 시점에 이미 커밋된 가장 최신 데이터의 버전을 $T$에게 제공
> - 일관된 읽기 (Consistent Read): 트랜잭션이 진행되는 동안 아무리 다른 트랜잭션이 데이터를 바꾸고 커밋하더라도, $T$는 처음 시작할 때 가졌던 스냅샷만 가져 일관성을 보장.
> - 버퍼링된 쓰기: 트랜잭션이 데이터를 수정(write)하면 실제 원본을 바로 고치는 것이 아니라, 자신만의 로컬 버퍼에 새 버전을 만들어 커밋하는 순간에만 다른 트랜잭션에게 공개됩니다.

스냅샷은 동시에 접근하는 한 데이터에 대한 **갱신 손실**을 방지하기 위해,<br/>
**first commmitter wins**를 따른다.

#### First Commiter Wins<br/>

1. 트랜잭션 $T_1$과 $T_2$가 동일한 데이터 $X$를 동시에 수정을 시도
2. T_1$이 먼저 Commit
3. 뒤이어 $T_2$가 커밋을 시도할 때, "$T_2$가 시작된 이후에 $X$가 $T_1$에 의해 변경되어 커밋되었다"는 것을 감지
4. 데이터베이스는 $T_2$를 강제로 롤백(Abort)

First update wins는 먼저 write 혹은 X-Lock을 얻은 트랜잭션이 우선권을 얻는 것


**References**<br/>
Database Systems, Abraham Silberschatz, Henry Korth and S. Sudarshan