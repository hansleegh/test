---
layout: default
title: dba
permalink: /dba/
---
# dba
## 온프레미스 → Azure 마이그레이션 아키텍처
온프레미스 환경에서 Azure로 마이그레이션 시에는 서비스 도메인 분리와 데이터 소유권 이전을 우선 설계한다. 기존 단일 DB 구조는 서비스별 전용 DB(Owned Schema)로 분리하며, 내부 조인·트랜잭션 기반 통합을 이벤트 중심 비동기 통합(Event Bus, CDC)으로 전환한다. 일관성은 강한 일관성(ACID) 대신 **최종 일관성(Saga, Outbox 패턴)**을 적용하고, 읽기/쓰기 모델을 CQRS로 분리한다.
데이터 이행은 초기 스키마 배포 후 기본 데이터 적재를 진행하고, **CDC(Change Data Capture)**를 이용해 온프레미스 DB의 변경 사항을 Azure 환경으로 실시간 복제한다. 이때 Kafka·Debezium 기반 CDC 파이프라인을 구성해 느슨한 결합을 유지한다. 트랜잭션은 2PC 대신 Saga를 적용하여 분산 환경의 성능 저하를 방지한다.
마이그레이션 절차는
1. Bounded Context 기준 서비스·DB 스키마 설계
2. 신규 Azure DB 생성 및 초기 적재
3. CDC 기반 실시간 동기화 가동
4. 트래픽 점진 전환(Read → Write)
5. 최종 스위치오버(온프레미스 Write 중단 후 완전 전환)
