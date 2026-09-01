---
title: "Cache: 동작 원리와 구조"
type: note
date: 2026-08-27
last_modified_at: 2026-09-01
---

## 캐시란 무엇인가
### 캐시의 역할
  - Cache Memory는 속도가 빠른 장치와 느린 장치 간의 속도 차에 따른 병목 현상을 줄이기 위한 고속 버퍼 메모리(SRAM 기반)를 의미함
  - Cache 메모리는 주기억 장치에서 자주 사용하는 프로그램과 데이터를 저장해두어 처리 속도를 빠르게 함
    - Cache 메모리가 어느정도 역할을 제대로 수행하기 위해서는 CPU가 어떤 데이터를 원할 것인가를 어느정도 예측할 수 있어야 함 -> 예측 정도가 캐시의 성능과 직결됨
### Locality: temporal / spatial
  - Locality(데이터 지역성)란, 프로그램이 기억 장치 내의 정보를 균일하게 Access 하는 것이 아닌 어느 한 순간에 특정 부분을 집중적으로 참조하는 특성을 의미함
  - Locality는 대표적으로 temporal/spatial으로 구분됨
    - Temporal Locality (시간적 지역성)
      - 최근에 참조된 주소의 내용은 곧 다음에 다시 참조됨
        - 예시: 메모리 상의 같은 주소에 여러 차례 R/W를 수행할 경우, 상대적으로 작은 크기의 cache를 사용해도 효율성을 꾀할 수 있음
    - Spatial Locality (공간적 지역성)
      - 기억장치 내에 서로 인접하여 저장되어 있는 데이터들이 연속적으로 참조될 가능성이 높아짐
      - CPU 또는 Disk Cache의 경우 한 메모리 주소에 접근할 때, 그 주소뿐 아니라 해당 블록을 전부 캐시에 가져오게 됨
        - 예시: 메모리 주소를 오름차순/내림차순으로 접근한다면, 캐시에 이미 저장된 같은 블록의 데이터를 접근하게 되므로 캐시의 효율성이 크게 향상됨

## 캐시 기본 동작
### 캐시 메모리 관련 용어
  - Block / Cache Line
    - Block: 캐시 메모리와 메인 메모리 사이에 주고받는 데이터의 단위
    - Cache Line: 그 block 하나를 담는 캐시 쪽 자리, `[ valid bit | dirty bit(optional) | tag | data(= block) ]` 구조
      - valid bit: 그 자리에 유효한 데이터가 들어있는지를 나타냄
      - dirty bit: 캐시에서 수정되었으나 아직 하위 계층에 반영되지 않은 내용이 있는지를 나타냄 (write-back 방식에서만 필요)
  - Hit/Miss (`Hit_rate + Miss_rate = 1`)
    - Hit: CPU가 읽어오려고 하는 데이터가 캐시에 있는 경우
      - Hit rate: CPU가 요청한 데이터 중 캐시에 저장된 비율
      - Hit time(latency): 캐시에서 데이터를 읽어오는 데 필요한 시간
    - Miss: CPU가 읽어오려고 하는 데이터가 캐시에 없는 경우
      - Miss rate: CPU가 요청한 데이터 중 캐시에 저장되지 않은 비율
      - Miss penalty(latency): miss가 발생해 데이터를 block만큼 다음 하위 계층(L2, 메인 메모리 등)에서 캐시 메모리로 가져오는 데 필요한 시간
  - AMAT(Average Memory Access Time)
    - CPU가 메모리에 접근할 때 걸리는 평균 시간을 의미함, 캐시 메모리의 성능 평가 지표
    - `AMAT = Hit_time + (Miss_rate * Miss_penalty)`
### 캐시 구성: Block offset / Index / Tag

<figure>
  <img src="/assets/images/cache-organization.png" alt="Cache organization with S sets, E lines per set, and B bytes per block (C = S x E x B)">
</figure>

<figure>
  <img src="/assets/images/cache-address-split.png" alt="Address split into tag, set index, and block offset, mapped onto a cache line">
</figure>

  - Memory address는 상위 bit부터 `[ Tag | Index | Block offset ]` 순서로 잘리며, 각 구간의 크기는 아래 공식으로 정해짐
  - Block offset 
    - 필요한 data에 접근하기 위해서는 Bytes 단위로 접근해야하며, `Block_size`Byte 중에서 몇 번째 Byte에 접근하는지 찾는 방법
    - `Offset_bits = log2(Block_size)`
  - Indexing
    - 접근하는 Memory address에 대해 Cache 안의 `Set_수`개의 set 중에서 어디에 대응되는지 찾는 방법
      - `Set_수 = Cache_size / (Block_size * N_way)` (`N_way`는 배치 정책이 정하는 값)
    - `Index_bits = log2(Set_수)`
  - Tag
    - Indexing과 Block offset 만으로는 서로 다른 Memory address끼리 같은 Cache 공간에 대응되는 상황이 발생할 수 있음
    - 지금 그 자리에 들어와 있는 주소가 무엇인지 구분하기 위해, 주소의 남은 상위 bit을 cache line마다 tag로 저장해두고 Tag Matching을 수행함
    - `Tag_bits = Address_bits - Index_bits - Offset_bits`
    - 주소에서 잘라낸 tag bit 수와 cache line이 실제로 들고 있는 관리 bit 수는 구분해야 함
      - 주소 쪽 tag = `Tag_bits`
      - Cache line 쪽 = `Tag_bits + 1(valid_bit) + 1(dirty_bit, optional)` 이 data와 함께 저장됨

### 캐시 접근 흐름 (Cache Access Flow)

<figure>
  <img src="/assets/images/4-way_set_associative.png" alt="4-way set associative cache datapath with tag comparators, hit signal, and data multiplexer">
</figure>

  - Lookup (Read/Write 공통)
    - 주소 분할: 요청된 memory address를 `[ Tag | Index | Block offset ]`으로 자름
    - Set 선택: Index bit으로 확인할 set 하나를 고름
    - Valid bit 확인: 해당 block에 유효한 데이터가 들어있는지 검사 (0이면 저장된 tag는 무의미하므로 곧바로 miss)
    - Tag Matching: address의 tag bit과 Cache에 있는 tag bit이 같은지 검사하는 것을 의미함
      - 선택된 set 안의 way들에 대해 병렬로 수행되며, valid == 1 이면서 tag가 일치하는 way가 하나라도 있으면 hit, 없으면 miss
    - Byte 선택: hit인 경우 그 way의 data(= block)에서 Block offset이 가리키는 위치의 byte를 골라 CPU로 넘김
  - Read
    - Read hit으로 판정되는 경우
      - 해당 way의 block에서 Block offset 위치의 데이터를 바로 읽어 CPU로 전달함 (하위 계층 접근 없이 Hit time만 소요됨)
      - 교체 정책이 참조하는 상태를 이때 갱신함 (LRU 계열에 한정되며, FIFO는 삽입 순서만 보므로 hit에 갱신하지 않고 Random은 상태 자체가 없음)
    - Read miss로 판정되는 경우 - Fill 과정이 진행됨
      - 다음 하위 계층(L2, 메인 메모리 등)에서 해당 block을 통째로 가져와 cache line을 채움
      - 들어갈 자리가 이미 차 있으면 교체 정책으로 victim을 선정함
        - 이때 victim이 dirty한 상태라면 그 시점에 write-back이 발생함
      - Fill이 끝나면 valid bit을 1로, tag를 해당 주소의 tag로 세팅한 뒤 hit과 동일하게 데이터를 읽어 CPU로 전달함
      - 즉 Read miss의 비용은 `Hit_time + Miss_penalty`이며, 이것이 AMAT 식의 근거가 됨
  - Write
    - Write hit으로 판정되는 경우 - 수정한 내용을 어느 계층까지 반영할 것인가의 문제 (write-through / write-back)
    - Write miss로 판정되는 경우 - 쓰기 전에 해당 block을 캐시로 가져올 것인가의 문제 (write-allocate / no-write-allocate)

## 배치 정책 (Placement / Mapping) — associativity
  - Mapping(사상)이란, 주기억장치의 block이 캐시의 어느 위치에 놓일 수 있는지를 결정하는 대응 규칙을 의미함
### 직접 매핑 (Direct-mapped) / 연관 매핑 (Fully-associative) / 집합 연관 매핑 (N-way set-associative)

<figure>
  <img src="/assets/images/cache_mapping.png" alt="Direct-mapped, set-associative, and fully-associative block placement compared">
</figure>

  - 직접 매핑 (Direct-mapped)
    - 가장 기본적인 캐시 메모리 구조
    - 각 memory block이 cache의 정확히 한 block에만 대응되는 구조
  - 연관 매핑 (Fully-associative)
    - Direct-mapped와 정반대의 캐시 메모리 구조
    - 각 memory block이 캐시 메모리의 모든 block 중 아무 곳에나 대응될 수 있는 구조
      - hit/miss 판정을 위해서 cache의 모든 block을 확인해야 함
  - 집합 연관 매핑 (N-way set-associative)
    - Direct-mapped와 Fully-associative의 중간의 캐시 메모리 구조
    - 캐시 메모리를 n개의 block(way)을 가진 set이라는 단위로 나누고, 각 memory block은 정해진 하나의 set 안에서는 아무 way에나 대응될 수 있는 방식
      - hit/miss 판정을 위해서 하나의 set에 존재하는 cache block(way)을 모두 확인해야 함
    - N-way tradeoff(N 값이 커질수록)
      - 장점: 같은 자리를 두고 다투던 block들이 분산되어 conflict miss가 줄고 hit rate가 오름
      - 단점: tag 비교기를 n개 병렬로 돌려야 해서 면적/전력이 늘고 hit time이 길어짐, LRU 등 교체 상태를 관리하는 비용도 함께 늘어남
      - 기타: 주소 분할 방식 변경됨, index bit이 줄고 그만큼 tag bit이 늘어남

## 캐시 교체(Replacement) 알고리즘
  - 캐시 크기의 제한으로 인하여 캐시에 모든 데이터를 저장할 수 없음
    - 따라서 캐시가 가득 찬 상태에서 새 block을 채워야 할 때, 캐시 교체 알고리즘에 따라 어떤 block(victim)을 버릴지 결정함
### 캐시 교체 알고리즘의 종류
  - LRU(Least Recently Used)
    - 가장 오랫동안 사용되지 않은 cache block을 교체하는 방식
    - Temporal locality를 그대로 반영한 방식이라 hit rate 측면에서 기준점이 됨
    - set 안 way들의 사용 순서를 저장하고 hit마다 갱신해야 하므로, way 수가 늘면 상태 bit과 갱신 회로가 급격히 커짐
  - pseudo-LRU (tree-PLRU)
    - 정확한 사용 순서를 포기하고 "대략 오래된 것"을 고르는 LRU의 근사 방식
    - way를 이진 트리의 잎에 두고, 각 내부 노드에 1bit씩만 두어 "이 갈림길에서 어느 쪽이 덜 최근에 쓰였는가"만 기억함
      - victim 선정: 루트에서 bit이 가리키는 방향을 따라 잎까지 내려감
      - hit 시 갱신: 지나온 노드의 bit을 반대 방향으로 뒤집음
    - N-way에 `N-1`bit만 필요하여 true LRU보다 훨씬 저렴함 (4-way 기준 3bit)
    - 가끔 진짜 LRU와 다른 victim을 고르지만 hit rate 손해가 작아, 실제 L1/L2 캐시에 널리 쓰임
  - FIFO(First In First Out)
    - 가장 먼저 들어온 cache block을 교체하는 방식
    - 삽입 순서만 보므로 hit에는 상태를 갱신하지 않아 LRU보다 저렴함
    - 자주 쓰이는 block도 오래됐다는 이유만으로 쫓겨날 수 있어 hit rate가 LRU보다 떨어짐
  - Random
    - 제거할 cache block을 임의로 선택하여 교체하는 방식
    - 관리할 상태가 아예 없어 가장 저렴하며, associativity가 큰 경우 LRU와의 hit rate 차이가 크지 않아 실제로 채택되기도 함
### 참고: OS 페이지 교체 알고리즘 (하드웨어 캐시와 구분할 것)
  - 아래 알고리즘들은 운영체제가 소프트웨어로 수행하는 페이지 교체 알고리즘으로, 하드웨어 캐시 교체에는 쓰이지 않음
    - 페이지 교체는 디스크 I/O를 앞두고 일어나 결정에 쓸 시간이 충분하고, 참조 횟수 같은 상태도 커널 입장에서는 그냥 변수라 비용이 낮음
    - 반면 하드웨어 캐시에서 카운터는 cache line마다 물리적으로 달아야 하는 부품이므로, 수천 개 line에 대해서는 면적·시간 모두 성립하지 않음
  - LFU(Least Frequently Used): 사용(참조) 횟수가 가장 적은 데이터를 교체하는 방식
    - MFU(참조 횟수가 가장 많은 데이터를 교체하는 방식) 알고리즘도 존재함
  - NUR(Not Used Recently): LRU와 유사하지만, 참조 bit과 변형 bit 2개만 사용하여 LRU의 오버헤드를 줄인 방식
  - SCR(Second Chance Replacement): FIFO 방식에 참조 bit을 추가하여, 참조된 페이지에는 한 번 더 기회를 주는 방식

## 쓰기 정책 (Write)
### 캐시 일관성(Cache Coherence)
  - 캐시에서 "데이터가 일치한다"는 말은 서로 다른 두 방향을 가리킬 수 있으므로 구분해야 함
    - 엄밀한 의미의 Cache Coherence는 **멀티 코어 간의 일관성**만을 의미함

| 구분 | 캐시 ↔ 하위 계층 | 멀티 코어 간의 일관성 |
|---|---|---|
| 무엇을 맞추는가 | 캐시의 내용과 메인 메모리의 내용 | 여러 코어가 각자의 캐시에 들고 있는 같은 주소의 복사본 |
| 언제 깨지는가 | 캐시에만 쓰고 하위 계층에 반영하지 않았을 때 | 한 코어가 자기 캐시의 복사본만 수정했을 때 |
| 해결 수단 | Write 정책<br>(write-through / write-back) | Coherence protocol<br>(MESI 등) |

### 캐시 메모리 Write 정책: write-through vs write-back
  - write-through
    - 캐시 메모리에 데이터가 Write되는 시점에 해당 데이터를 메인 메모리에도 저장하는 정책 (캐시 메모리와 실제 메모리 저장소 모두에 데이터를 업데이트 하는 정책)
    - 장점: 캐시메모리와 하위계층 메모리 간의 일관성을 유지할 수 있어서 안정적임
    - 단점: 처리속도가 느림 (보완책: 캐시 메모리에 write buffer를 추가하여 사용함)
  - write-back(lazy write)
    - 캐시 메모리에만 데이터를 write하고 있다가, 새로운 데이터 블록으로 교체되는 시점에 해당 데이터를 메인 메모리에도 저장하는 정책
      - dirty bit(status bit)의 수명: write hit이 발생하면 1로 세팅되고, 그 line이 victim으로 선정되어 하위 계층에 반영(write-back)된 뒤 0으로 클리어됨
      - 즉 dirty bit은 "이 line의 최신 값이 캐시에만 있다"는 표시이며, eviction 시 write-back이 필요한지를 이 bit 하나로 판정함 (clean이면 그냥 덮어씀)
    - 장점: write-through 정책보다 속도가 빠름
    - 단점: 구현이 어려움(최신 값이 캐시에만 존재하는 구간이 생겨 Coherence protocol이 복잡해짐)
### Write miss 발생 시의 정책: write-allocate vs no-write-allocate
  - Read miss는 CPU에게 값을 넘겨주어야 하므로 무조건 block을 캐시로 가져와야 하지만, write miss는 CPU가 값을 돌려받을 필요가 없으므로 "굳이 캐시로 가져올 것인가"라는 선택지가 생김
  - write-allocate (fetch-on-write)
    - write miss가 발생하면 해당 block을 하위 계층에서 캐시로 가져온 뒤(Read miss와 동일한 Fill 절차), 그 위에 데이터를 write함
    - CPU가 쓰려는 것은 block의 일부(수 byte)뿐인데도 block 전체를 가져오는 이유는, 나머지 부분까지 채워져야 그 cache line을 valid로 둘 수 있기 때문임
    - 쓴 자리를 곧 다시 읽거나 쓰는 경우(locality가 있는 경우)에 유리함
  - no write-allocate (write around)
    - write miss가 발생하면, 캐시 메모리에 새로운 데이터 블록을 로드하지 않고 직접 메모리에 write함 (캐시 상태는 그대로 유지됨)
    - 한 번 쓰고 다시 참조하지 않는 데이터(큰 버퍼를 순차적으로 채우는 경우 등)에 유리하며, 쓰이지 않을 block이 기존 block을 밀어내는 cache pollution을 막을 수 있음
  - 통상적인 조합
    - write-back + write-allocate: 일단 캐시에 올려두면 이후 같은 block에 대한 write가 모두 캐시에서 흡수되므로, 하위 계층 트래픽을 줄인다는 두 정책의 목적이 서로 맞음
    - write-through + no write-allocate: 어차피 모든 write가 하위 계층으로 나가므로, 캐시에 block을 채워도 트래픽이 줄지 않아 Fill 비용을 치를 이유가 없음

## 캐시 계층 구조
### L1/L2/L3 Cache

<figure>
  <img src="/assets/images/cache-hierarchy.png" alt="Cache hierarchy of a multicore processor with per-core L1 i/d and L2, and a shared L3">
</figure>

  - 캐시의 계층이 낮을수록 빠르고 용량이 작으며, 높을수록 느리고 용량이 큼
    - L1 캐시: 가장 빠른 캐시이며, CPU 파이프라인과 직접 연결되어 있어 매 사이클 접근 가능함
      - L1은 명령어용(L1-I)과 데이터용(L1-D)으로 분리되어 있음 (Harvard 구조)
        - 파이프라인에서 명령어 fetch와 데이터 load/store가 같은 사이클에 발생하므로, 하나로 합쳐두면 매 사이클 포트 경합이 일어남
        - 둘로 나누면 각각 1포트만으로 동시 처리가 가능하고, 접근 패턴이 서로 다른(명령어는 순차적, 데이터는 불규칙) 두 스트림이 서로를 밀어내지 않는 이점도 있음
      - L2 이하는 L1이 대부분의 요청을 걸러내어 접근 빈도가 낮으므로, 경합이 문제되지 않아 통합(unified) 캐시로 둠
    - L2 캐시: L1보다 크고 느리며, 보통 코어별로 전용으로 할당됨
    - L3 캐시(LLC, Last Level Cache): 여러 코어가 공유하는 대용량 캐시, 멀티코어 환경에서 코어 간 데이터 공유를 지원함

| 캐시 | L1-I(명령어) | L1-D(데이터) | L2 | L3 |
|---|---|---|---|---|
| 용량 (예시) | 32~64KB/코어 | 32~64KB/코어 | 256KB~1MB/코어 | 8~64MB |
| Latency | ~1ns (4사이클) | ~1ns (4사이클) | ~4ns (12사이클) | ~(10~20)ns (40사이클) |
| 위치 | 코어 내부 | 코어 내부 | 코어 내부 | 칩 내부 |
| 코어 간 공유 여부 | 코어 전용 | 코어 전용 | 코어 전용 | 모든 코어 공유 |

### 다계층 캐시의 성능 계산
  - 계층이 여러 개일 때, `캐시 기본 동작`의 AMAT 식은 재귀적으로 확장됨
    - 2계층: `AMAT = Hit_time + (Miss_rate * Miss_penalty)`
    - 다계층: 상위 계층의 Miss_penalty 자리에 그 아래 계층의 AMAT이 그대로 들어감
      - `AMAT = L1_hit + L1_miss_rate(local) * (L2_hit + L2_miss_rate(local) * (L3_hit + L3_miss_rate(local) * Mem_access_time))`
    - 계층을 하나씩 더 둘 때마다 상위 계층의 miss penalty가 줄어듦
  - Local miss rate vs Global miss rate

| 구분 | Local miss rate | Global miss rate |
|---|---|---|
| 분모 (무엇에 대한 비율인가) | **그 계층에 도달한** 요청 수 | **CPU가 낸 전체** 요청 수 |
| L2를 예로 들면 | L2까지 내려온 요청 중 L2가 놓친 비율 | 전체 요청 중 L2까지도 잡지 못한 비율 |

  - `L2_global_miss_rate = L1_miss_rate * L2_local_miss_rate`
  - 하위 계층일수록 local miss rate가 높게 나옴 -> hit하기 쉬운 요청은 이미 상위 계층에서 걸러지고, hit하기 어려운 요청만 내려오기 때문

### private vs shared
  - 같은 계층을 코어마다 따로 둘 것인지(private), 여러 코어가 함께 쓸 것인지(shared)의 문제
    - 통상 L1/L2는 private, L3(LLC)는 shared로 구성함
  - 아래 계층으로 갈수록 shared가 유리해지는 이유
    - 접근 빈도가 낮아 지연이 늘어나는 손해가 작음
    - 용량이 커서 코어 간에 나눠 쓸 때의 활용률 이득이 큼

| 구분 | private (코어 전용) | shared (코어 공유) |
|---|---|---|
| 접근 지연 | 코어 옆에 붙어 있어 짧음 | 거리가 멀고 중재가 필요하여 김 |
| 용량 활용 | 한 코어가 놀아도 그 용량을 다른 코어가 쓸 수 없음 | 필요한 코어가 더 많이 가져다 쓸 수 있음 |
| 데이터 공유 | 같은 데이터를 코어마다 중복 보관 | 사본 하나를 여러 코어가 공유 |
| 일관성 | 사본이 여러 개이므로 coherence 관리 대상이 됨 | 사본이 하나이므로 그 계층 안에서는 문제되지 않음 |

### Inclusive / Exclusive / NINE
  - 한 block 데이터를 여러 계층에서 중복해서 들고 있을 것인가에 대한 규칙

| 정책 | Inclusive | Exclusive | NINE (Non-Inclusive Non-Exclusive) |
|---|---|---|---|
| 규칙 | 하위 계층(L1/L2)에 있는 block은 상위 계층(L3)에도 반드시 존재함 | 한 block은 한 계층에만 존재함 | 둘 중 어느 것도 강제하지 않음 |
| 유효 용량 | L3 용량이 사실상 전부 | L1 + L2 + L3의 합 | 그 중간 |
| 코어 간 사본 탐색 | 공유 L3만 검사하면 됨<br>(L3에 없으면 어느 코어의 L1/L2에도 없음이 보장됨) | 모든 코어의 L1/L2를 일일이 확인해야 함 | 포함 관계가 보장되지 않아 별도의 스누프 필터가 필요할 수 있음 |
| 대가 | back-invalidation(L3에서 쫓겨난 block은 하위 계층에서도 함께 무효화되므로, 해당 코어는 잘 쓰던 데이터를 잃음) | 계층 간에 block을 주고받는 이동(swap)이 잦아 관리가 복잡해짐 | 동작이 상황에 따라 달라져 분석이 어려움 |

## References
- [[OS] 캐시 메모리(Cache Memory)란? 캐시의 지역성(Locality)이란?](https://chelseashin.tistory.com/43)
- [캐시의 지역성(Cache Locality)](https://velog.io/@lob3767/%EC%BA%90%EC%8B%9C%EC%9D%98-%EC%A7%80%EC%97%AD%EC%84%B1Cache-Locality)
- [[컴퓨터구조] Cache](https://inyongs.tistory.com/134)
- [[컴퓨터구조] 캐시 메모리](https://velog.io/@letskuku/%EC%BB%B4%ED%93%A8%ED%84%B0%EA%B5%AC%EC%A1%B0-%EC%BA%90%EC%8B%9C-%EB%A9%94%EB%AA%A8%EB%A6%AC)
- [기본 cache memory 구조](https://mydailylogs.tistory.com/30)
- [[컴퓨터구조] 캐시 메모리의 동작원리, 종류, Replacement Policy](https://m.blog.naver.com/msh5420/223512280258)
- [[Cache] 캐시 동작과 캐시 교체 정책](https://gengmi.tistory.com/entry/Cache-%EC%BA%90%EC%8B%9C-%EB%8F%99%EC%9E%91%EA%B3%BC-%EC%BA%90%EC%8B%9C-%EA%B5%90%EC%B2%B4-%EC%A0%95%EC%B1%85)
- [캐시 교체알고리즘과 상황별 효율성](https://hpotter1993.tistory.com/54)
- [캐시 메모리의 쓰기 정책: Write-Through, Write-Back](https://blog.naver.com/PostView.nhn?blogId=techref&logNo=222282586535)
- [[컴퓨터구조] 캐시 메모리 완벽 정리](https://wolleyneerg.tistory.com/69)
- [실제 캐시 계층 구조의 해부](https://mydailylogs.tistory.com/34)
