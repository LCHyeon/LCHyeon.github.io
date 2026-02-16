---
title: 메시지 브로커와 Kafka - 2
description: Redis, RabbitMQ와의 쓰임새 비교를 통해 Kafka를 실제 시스템에서 사용하는 방식 정리
author: lucas
date: 2026-02-16 10:00:00 +0900
categories: [Backend, Architecture]
tags: [message-broker, message-queue, pubsub, architecture]
pin: false
math: false
mermaid: false
---

## 1. Redis, RabbitMQ, Kafka의 차이점

분산 시스템에서 비동기 처리를 고민하면 다음 세 가지가 자주 후보로 등장한다.

- Redis  
- RabbitMQ  
- Kafka  

셋 다 메시지를 전달할 수 있지만 실제 개발에서의 쓰임새는 다르다.

| 구분 | Redis | RabbitMQ | Kafka |
|---|---|---|---|
| 주 사용 목적 | 빠른 데이터 전달 / 캐시 | 비동기 작업 처리 | 이벤트 저장 및 데이터 흐름 관리 |
| 메시지 성격 | 빨리 보내면 끝 | 처리되면 끝 | 기록으로 남김 |
| 데이터 보존 | 거의 없음 | 소비 시 제거 | 일정 기간 유지 |
| 대표 사용 예 | 채팅, 알림, 세션 | 이메일 발송, 이미지 처리, 백그라운드 작업 | 로그 수집, 이벤트 스트림, 실시간 분석 |

Redis나 RabbitMQ는 **메시지를 전달하기 위해** 사용된다면, Kafka는 보통 **시스템의 이벤트 흐름을 만들기 위해** 사용된다.

---

## 2. Kafka는 실제 개발에서 어떻게 쓰는가

Kafka는 메시지 전달 시스템이라기보다 **서비스에서 발생하는 이벤트를 기록하고 활용하는 시스템**으로 사용된다.

실제 개발에서는 보통 세 가지 방식으로 등장한다.


### 2.1 서비스 이벤트 기록
---

가장 흔한 Kafka 사용 패턴이다.

서비스에서 중요한 이벤트가 발생하면 DB 저장과 동시에 Kafka에도 기록한다.

```
서비스 → DB 저장
        → Kafka 이벤트 기록
```

예:

- 회원 가입 이벤트
- 주문 생성 이벤트
- 결제 완료 이벤트
- 사용자 행동 로그

코드에서는 보통 이런 형태가 된다.

```java
kafkaTemplate.send("user.created", event);
```

이렇게 기록된 이벤트는 이후

- 로그 분석
- 통계 집계
- 추천 시스템
- 알림 서비스

에서 다시 사용된다.

Kafka는 이 경우 **이벤트 저장소** 역할을 한다.


### 2.2 서비스 간 비동기 통신
---

Kafka는 서비스 간 직접 API 호출을 줄이기 위해 사용된다.

```
Order Service → Kafka → Payment Service
                      → Notification Service
                      → Analytics Service
```

예:

주문 서비스가 주문 생성 이벤트를 Kafka에 기록하면 각 서비스가 해당 이벤트를 소비한다.

- 결제 서비스 → 결제 처리
- 알림 서비스 → 문자 발송
- 분석 시스템 → 통계 반영

코드에서는 Consumer가 이벤트를 구독하는 방식이 된다.

```java
@KafkaListener(topics = "order.created")
public void handleOrderCreated(OrderEvent event) {
    // 결제 처리 로직
}
```

이 구조를 사용하면

- 서비스 간 결합도가 줄어들고
- 장애 전파가 감소하며
- 시스템 확장이 쉬워진다.


### 2.3 데이터 파이프라인 구성
---

Kafka는 단순 메시지 전달보다  
**데이터 흐름의 중심**으로 사용되는 경우가 많다.

```
서비스 로그 → Kafka → 실시간 처리 → 저장소
```

예:

- 서비스 로그 수집
- 사용자 행동 데이터 분석
- 실시간 통계 처리
- 검색 인덱스 업데이트

Kafka는 이 경우

- 데이터 수집 버퍼
- 스트림 처리 입력
- 이벤트 기록 저장소

역할을 동시에 수행한다.

---

## 핵심 정리

> - Redis는 빠른 전달과 캐시에 강하다  
> - RabbitMQ는 작업 전달에 강하다  
> - Kafka는 이벤트 저장과 데이터 흐름 관리에 강하다  
{: .prompt-tip }

---

## 참고 자료

1. [Kafka 공식 문서](https://kafka.apache.org/documentation/#introduction)  
2. [Kafka 등장 배경과 사용방법](https://velog.io/@tess/kafka-%EC%82%AC%EC%9A%A9-%EC%A0%95%EB%A6%AC) 
