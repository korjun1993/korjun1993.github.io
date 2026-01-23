---
layout: post
title: 엘라스틱서치 분산 아키텍처
categories: elasticsearch
description: elasticsearch
keywords: elasticsearch
---

## 원본 자료

---

- https://www.elastic.co/docs/deploy-manage/distributed-architecture/clusters-nodes-shards/node-roles
- https://www.elastic.co/docs/deploy-manage/distributed-architecture/discovery-cluster-formation
- https://www.elastic.co/docs/deploy-manage/distributed-architecture/discovery-cluster-formation/modules-discovery-quorums
- https://www.elastic.co/docs/deploy-manage/distributed-architecture/discovery-cluster-formation/modules-discovery-voting



## 목차

---

- 원본 자료
- 분산 아키텍처
  1. [Discovery](#Discovery)
  2. [시드 호스트 공급자 (Seed hosts providers)](#시드-호스트-공급자-seed-hosts-providers)
  3. [Quorum 기반 의사결정](#quorum-기반-의사결정)
  4. [투표 구성 (voting configuration)](#투표-구성-voting-configuration)
  5. [부트스트랩 구성(bootstrap configuration)](#부트스트랩-구성bootstrap-configuration)
  6. [실무 운영 가이드](#실무-운영-가이드)

## 분산 아키텍처

---

### Discovery

- 클러스터 형성 모듈(cluster formation module)이 다른 노드를 찾는 과정 
- 시드 호스트 공급자로부터 받은 시드 주소 목록과, 마지막으로 마스터 후보로 알려졌던 주소를 대상으로 다음 두 단계 실행
  - probing: 각 노드는 전달받은 시드 주소로 연결을 시도하여 연결된 노드가 무엇인지 식별하고, 해당 노드가 마스터 후보 자격이 있는지 확인
  - peer sharing: 연결에 성공한 노드들은 자신이 알고 있는 모든 마스터 후보 목록(peer) 목록을 공유함. 그 후 노드는 원격 노드로부터 전달받은 피어 목록을 대상으로 다시 프로빙을 수행하고, 피어 목록을 공유하는 과정을 반복함. 결과적으로 도달 가능한 경로상에 있는 모든와 통신하게 됨.
- `discovery.find_peers_interval` (기본값=1초) 간격으로 디스커버리 수행
- 상황별 조건을 만족할 때까지 디스커버리 수행
  - 클러스터 첫 시작일 때 → 부트스트랩 구성의 과반수 이상의 노드가 서로를 발견할 때까지
  - 클러스터가 재시작일 때 → 각 마스터 후보 노드의 디스크에 저장된 투표 구성 중 최신 버전을 기준으로 과반수 이상 노드를 찾을 때 까지



### 시드 호스트 공급자 (Seed hosts providers)

- 클러스터 형성 모듈은 시드 목록을 설정하기 위해 기본적으로 설정 기반과 파일 기반, 두 가지 공급자를 제공
- 각 공급자는 시드 노드의 IP 주소나 호스트네임을 제공 (호스트네임은 DNS 조회를 통해 IP로 변환)
- 포트 번호를 명시하지 않으면 `transport.profiles.default.port` 또는 `transport.port`에 설정된 포트 범위를 사용

설정 예시 (discovery.seed_hosts)

```
discovery.seed_hosts:
- search-es-live-01
- search-es-live-02
- search-es-live-03
- search-es-live-04
- search-es-live-05
- search-es-client-live-01
- search-es-client-live-02
```


### Quorum 기반 의사결정

- 클러스터 상태를 변경하는 모든 결정은 선출된 단 한 대의 마스터 노드(elected master)를 통해서 이루어짐
- 단, 결정을 내리기 위해 클러스터 내 마스터 후보들의 과반수 이상의 집합인 쿼럼(Quorum)의 동의가 필요
- 인덱스 관리(생성, 삭제, 매핑 변경), 샤드 재배치 명령 등의 API는 마스터 노드가 쿼럼의 동의를 구하여 실행
- 문서 작업(색인, 업데이트, 삭제), 조회, 집계 API는 코디네이팅 노드가 직접 데이터 노드와 통신하여 실행
- 만약, 마스터 선출 과정에 과반수 조건이 없는 경우, 마스터 노드가 두 개 선출되어 split-brain 현상이 발생할 수 있음. 쿼럼 기반 의사 결정을 통해 두 명의 마스터가 나올 수 없도록 강제하여 split-brain 현상을 방지
- 노드가 추가/제거될때마다 Elasticsearch는 투표에 참여할 명단(voting configuration)을 자동으로 업데이트
- 노드 전체가 아닌 일부의 응답만 요구함으로써, 일부 노드에 장애가 생겨도 클러스터 중단 없이 작업을 계속 할 수 있음
- 만약 절반 이상의 노드를 동시에 중단하면, 과반수 이상의 노드가 다시 온라인 상태가 될 때까지 클러스터를 사용할 수 없음
- 노드가 3대일 때나 4대일 때나 견딜 수 있는 장애 노드 수(fault tolerance)는 1대로 동일. 따라서 엘라스틱서치는 비용 대비 효율성과 명확한 과반수 형성을 위해 마스터 후보 노드가 짝수인 경우, 1대를 투표에서 제외하여 홀수 쿼럼을 유지함

> - 쓰기 충돌 - {id: 1, name:"gemini"} 수정 요청 → A 마스터에게 전달. 동시에 다른 유저가 {id: 1, name:"ChatGPT"} 수정 요청 → B 마스터에게 전달. 네트워크가 복구되어 두 진영이 다시 합쳐질 때, 데이터 충돌 발생 
> - 인덱스 매핑 및 설정 충돌 - integer age 필드 생성 → A 마스터에게 전달. keyword age 필드 생성 → B 마스터에게 전달. 클러스터가 합쳐질 때, 하나의 인덱스 안에 같은 필드가 서로 다른 타입으로 공존하여 시스템 오류발생 or 특정 진영의 인덱스 데이터가 손실 
> - 샤드 할당 및 데이터 유실 - A 마스터는 B 진영의 노드들이 죽었다고 판단하고, 비어있는 자기 쪽 노드들에 복제본(Replica) 샤드를 Primary로 승격. 같은 시각, B 마스터도 똑같은 판단을 내림. 이 후, 나눠진 클러스터에 쓰기 작업이 발생하고, 네트워크가 복구되면, 똑같은 번호의 Primary 샤드 중 한쪽을 버려야 함. 버려지는 쪽 진영에서 그동안 쌓였던 데이터는 영구적으로 유실 </br>

### 투표 구성 (voting configuration)

- 새로운 마스터를 선출하거나 새로운 클러스터 상태를 확정(commit)할 때, 그 응답을 카운트하는 마스터 후보 노드들의 집합
- 노드가 클러스터에 참여하거나 이탈하면, Elasticsearch는 투표 구성을 자동으로 조정
- 투표 구성이 변경되거나 클러스터 상태가 업데이트될때마다, 모든 마스터 후보 노드는 투표 구성을 디스크에 기록
- 노드를 종료시킬 때는, 한 대씩 차례대로 진행해야 함. 그렇지 않은 경우 교착 상태에 빠짐
  1. 5대 노드 중 1대의 노드를 종료 
  2. 클러스터에서 투표 구성을 4대로 업데이트 시도 
  3. 기다리지 않고, 관리자가 2대의 노드를 곧바로 종료 
  4. 투표 구성 변경 결정을 위한 쿼럼 불충분 → 교착 상태



### 부트스트랩 구성(bootstrap configuration)

- 완전히 새 클러스터가 처음 시작될 때 첫 번째 마스터를 선출하기 위한 투표 구성을 부트스트랩 구성(bootstrap configuration)이라고 함
- 각 노드는 랜덤 타이머에 의해 선거를 제안하며, 가장 먼저 선거를 제안하고, 과반수의 동의를 얻어낸 노드가 마스터로 선출됨
- 부트스트랩 구성은 클러스터 생성 시 단 한 번만 사용되며, 이후에는 디스크에 저장된 투표 구성을 보고 마스터를 뽑음
- 만약, 노드들이 각자 다른 initial_master_nodes를 바라볼 경우, 서로 다른 클러스터가 형성될 수 있음

예시

```
cluster.initial_master_nodes:
- search-es-live-01
- search-es-live-02
- search-es-live-03
```

### 실무 운영 가이드

- Rolling Restart: "노드 유지보수 시 한 대씩 껐다 켜는 이유는, 투표 구성이 자동으로 축소(Auto-shrink)되어 쿼럼 기준이 재조정될 시간을 벌어주어 가용성을 계속 유지하기 위함이다."
- Dedicated Master의 필요성: "쿼럼 기반 의사결정이 데이터 노드의 높은 부하(GC, CPU 집중 작업)에 방해받지 않도록 실무에서는 마스터 역할을 전용 노드로 분리한다."



