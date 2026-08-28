---
title: "Cache: 동작 원리와 구조"
type: note
date: 2026-08-27
last_modified_at: 2026-08-28
---

## 캐시란 무엇인가
### 캐시의 역할
  - Cache Memory는 속도가 빠른 장치와 느린 장치 간의 속도 차에 따른 병목 현상을 줄이기 위한 범용 메모리를 의미함
  - Cache 메모리는 주기억 장치에서 자주 사용하는 프로그램과 데이터를 저장해두어 처리 속도를 빠르게 함 (이 기능이 Cache Filtering이라고 표현되는 듯?)
    - Cache 메모리가 어느정도 역할을 제대로 수행하기 위해서는 CPU가 어떤 데이터를 원할 것인가를 어느정도 예측할 수 있어야 함 -> 예측 정도가 캐시의 성능과 직결됨
### Locality: temporal / spatial
  - Locality(데이터 지역성)란, 프로그램이 기억 장치 내의 정보를 균일하게 Access 하는 것이 아닌 어느 한 순간에 특정 부분을 집중적으로 참조하는 특성을 의미함
  - Locality는 대표적으로 temporal/spatial으로 구분됨
    - Temporal Locality (시간적 지연성)
      - 최근에 참조된 주소의 내용은 곧 다음에 다시 참조됨
        - 예시: 메모리 상의 같은 주소에 여러 차례 R/W를 수행할 경우, 상대적으로 작은 크기의 cache를 사용해도 효율성을 꾀할 수 있음
    - Spatial Locality (공간적 지연성)
      - 기억장치 내에 서로 인접하여 저장되어 있는 데이터들이 연속적으로 참조될 가능성이 높아짐
      - CPU 또는 Disk Cache의 경우 한 메모리 주소에 접근할 때, 그 주소뿐 아니라 해당 블록을 전부 캐시에 가져오게 됨
        - 예시: 메모리 주소를 오름차순/내림차순으로 접근한다면, 캐시에 이미 저장된 같은 블록의 데이터를 접근하게 되므로 캐시의 효율성이 크게 향상됨

## 캐시 기본 동작
### 캐시 메모리 관련 용어
  - Block (Cache Line): 캐시 메모리와 메인 메모리 사이에 주고받는 데이터의 단위
  - Hit/Miss (`Hit_rate + Miss_rate = 1`)
    - Hit: CPU가 읽어오려고 하는 데이터가 캐시에 있는 경우
      - Hit rate: CPU가 요청한 데이터 중 캐시에 저장된 비율
      - Hit time(latency): 캐시에서 데이터를 읽어오는 데 필요한 시간
    - Miss: CPU가 읽어오려고 하는 데이터가 캐시에 없는 경우
      - Miss rate: CPU가 요청한 데이터 중 캐시에 저장되지 않은 비율
      - Miss penalty(latency): miss가 발생해 데이터를 block만큼 메인 메모리에서 캐시 메모리로 가져오는 데 필요한 시간
  - AMAT(Average Memory Access Time)
    - CPU가 메모리에 접근할 때 걸리는 평균 시간을 의미함, 캐시 메모리의 성능 평가 지표
    - `AMAT = Hit_time + (Miss_rate * Miss_penalty)`
### 캐시에서 Data Read하는 방식: Indexing / Block offset / Tag
  - 아래 설명은 Block size (32B), Cache size (32KB), Memory Address (32bit)에서의 상황을 가정함
  - Indexing
    - 접근하는 Memory address에 대해 Cache 안의 1024개의 공간 중에서 어디에 있는지 찾는 방법
    - 1024 = 2^10이므로 10bit가 필요함
  - Block offset
    - 필요한 data에 접근하기 위해서는 Bytes 단위로 접근해야하며, 32 Byte 중에서 몇 번째 Byte에 접근하는지 찾는 방법
    - 32 = 2^5이므로 5bit가 필요함
  - Tag
    - 
### 읽기 흐름: lookup → hit/miss → fill

## 배치 정책 (Placement / Mapping) — associativity
  - Mapping(사상)이란, 캐시 기억장치와 주기억장치 사이에서 정보를 옮기는 것을 의미함
### 직접 매핑 (Direct-mapped) / 연관 매핑 (Fully-associative) / 집합 연관 매핑 (Set-associative, N-way)
  - 직접 매칭 (Direct Mapping)
    - 주기억장치의 블록들이 한 개의 캐시 라인으로만 사상될 수 있는 매핑 방법
### 세 방식 비교
### 같은 용량에서 way 수를 늘리면 무엇이 좋아지고 나빠지는가

## 교체 정책 (Replacement)
### LRU / pseudo-LRU / random / FIFO

## 쓰기 정책 (Write)
### write-through vs write-back
### write-allocate vs no-write-allocate

## 3Cs 다시 보기
### 각 miss가 용량, associativity, block size와 어떻게 대응되는가
### 개선 방안 → 어떤 C가 줄어드는가
### 예제: stride가 2의 거듭제곱일 때 conflict miss가 터지는 이유

## 계층 구조
### L1/L2/L3, inclusive/exclusive, latency·크기
### (선택) 멀티코어: private vs shared

## References
  - https://velog.io/@lob3767/%EC%BA%90%EC%8B%9C%EC%9D%98-%EC%A7%80%EC%97%AD%EC%84%B1Cache-Locality
  - https://chelseashin.tistory.com/43
  - https://velog.io/@letskuku/%EC%BB%B4%ED%93%A8%ED%84%B0%EA%B5%AC%EC%A1%B0-%EC%BA%90%EC%8B%9C-%EB%A9%94%EB%AA%A8%EB%A6%AC
  - https://inyongs.tistory.com/134