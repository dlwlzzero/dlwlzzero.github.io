---
title: "[Review] Roofline: An Insightful Visual Performance Model for Multicore Architectures"
date: 2026-08-26
last_modified_at: 2026-08-27
---

## Background
### Recent switch to multicore
  - ISA(Instruction Set Architecture, 명령어 집합 구조)의 다양화

    | CISC=Complex Instruction Set Computer | RISC=Reduced Instruction Set Computer |
    | ------------- | ------------- |
    | 가변 길이 명령어 | 고정 길이 명령어 |
    | pipelining **어려움** | pipelining **쉬움** |
    | 메모리 오퍼랜드 직접 사용<br>(연산 명령어가 메모리를 직접 읽고 씀)| Load/Store 구조<br>(메모리 접근은 `lw/sw`만, 나머지 연산은 레지스터끼리만) |
    | 마이크로코드 있음<br>(복잡한 명령어를 칩 내부의 작은 프로그램이 해석해서 실행)| 마이크로코드 없음<br>(hardwired 제어)|
    | 복잡도가 **하드웨어**에 있음 | 복잡도가 **컴파일러**에 있음 | 
    | `ADD [eax + ebx*4], 5`| `sll   $t0, $s1, 2      # ebx*4`<br>`add   $t0, $t0, $s0    # + eax → 주소`<br>`lw    $t1, 0($t0)      # 메모리 → 레지스터`<br>`addi  $t1, $t1, 5      # 레지스터에서 연산`<br>`sw    $t1, 0($t0)      # 레지스터 → 메모리`|
    | x86,VAX,68000 | MIPS,SPARC,ARM,RISC-V|

  - Cache vs Local Memory
 
    | 항목 | Cache | Local store/Scratchpad |
    | ------------- | ------------- | ------------- |
    | 데이터 이동 주체 | 하드웨어가 자동 (miss 시 fill, coherence 전부 처리) | 소프트웨어가 명시적으로 DMA/버퍼 관리 |
    | 주소 공간 | 메인 메모리와 동일 | 별도의 독립 주소 공간 |
    | 하드웨어 비용 | SRAM + 태그 배열 + 비교기 + 교체 로직 + coherence (비쌈) | SRAM Only (저렴) |
    | 접근 지연 | 예측 불가 (hit/miss에 따라 가변) | 항상 고정 -> 사이클 단위 스케줄링 가능 |
    | 적합한 워크로드 | 접근 패턴이 불규칙/동적인 범용 코드 | 컴파일 타임에 패턴을 아는 규칙적 연산 |
    | 대표 사례 | 범용 CPU의 L1/L2/L3 Cache | CUDA shared memory, Hexagon VTCM |
    
### Memory Wall: 왜 메모리가 병목인가
  - 연산 성능과 메모리 성능의 개선 속도가 수십 년째 벌어져 옴

    | 항목 | 연간 개선률 (경험적 추세) | 결과 |
    | ------------- | ------------- | ------------- |
    | 프로세서 연산 성능 | 약 50%/년 (2000년대 중반 이후엔 코어 수 증가로 지속) | 매년 Flop/s는 계속 늘어남 |
    | Off-chip DRAM 대역폭 | 약 25%/년 | 연산 대비 상대적으로 계속 뒤처짐 |
    | Off-chip DRAM 지연(latency) | 약 7%/년 | 격차가 가장 큼. prefetch/멀티스레딩으로 숨겨야 함 |

  - 그 결과, 칩이 낼 수 있는 Byte/Flop 비율이 해마다 낮아짐
    - 같은 코드라도 시간이 지날수록 연산기가 아니라 메모리에서 굶주리게 됨
    - 멀티코어는 코어 수만 늘릴 뿐 off-chip 핀 대역폭은 공유 -> 코어당 대역폭은 오히려 감소

### Insight over Accuracy: The 3Cs Model for Caches
  - 3Cs model for caches (`Total Cache Miss = Compulsory + Capacity + Conflict`)
    - Cache miss가 왜 발생하는지를 3가지 원인으로 분해하는 모델 (by Mark Hill)
   
      | 항목 | Compulsory<br>(cold miss) | Capacity | Conflict |
      | ------------- | ------------- | ------------- | ------------- |
      | 정의 |  그 블록에 처음 접근할 때, 캐시가 아무리 커도 데이터는 한 번은 메모리에서 올라와야 하므로 피할 수 없음 | 프로그램이 그 구간에서 필요로 하는 워킹셋이 캐시 용량보다 커서, 한 번 올라왔던 블록이 쫓겨났다가 다시 필요해지는 경우 | 용량은 남는데 같은 set이 매핑되는 블록이 associativity보다 많아서 쫓겨나는 경우. Fully-associativy였다면 안 났을 miss |
      | 측정 방법 | 무한 크기 fully-associative 캐시의 미스 수 | (크기 C의 fully-associative 미스) - Compulsory | (실제 캐시 미스) - (같은 크기 fully-associative 미스) |
      | 대안책 | 블록 크기 증가, HW/SW prefetch | 캐시 용량 증가, tiling/blocking으로 워킹셋 축소 | associativity 증가, victim cache, 배열 padding |
  - What the 3Cs model ignores
    - block size, block-allocation/replacement policy 등을 무시함 -> 용량과 associativity만 고려함

### Performance Models
  - predict program performance: stochastic analytical models, statistical performance models
  - **bound and bottleneck analysis**: Amdahl's Law (병렬 컴퓨터의 성능 향상은 병렬 프로그램의 직렬(순차) 부분에 의해 제한됨)
 
## Roofline Model이란 무엇인가?
  - 주어진 컴퓨팅 환경에서 성능의 제약조건이 어느 부분인지 쉽게 시각화할 수 있는 도구
  - 컴퓨팅 성능은 주로 ①메모리 대역폭과 ②연산기(최대 연산 능력)에 영향을 받으며, Roofline model에서는 이를 하나의 그래프에 시각화

## Roofline Model 뜯어보기
### Operational Intensity
  - 정의: `Flops / Byte`, DRAM에서 가져온 바이트당 몇 번의 부동소수점 연산을 하는지를 의미
    - 초당 연산량(Flop/s)이 아니라 **비율(ratio)**임에 주의
  - 측정 방법: caches와 메모리 사이의 traffic을 측정하면 됨
    - 커널이 반복 사용하는 데이터가 온칩 캐시 용량보다 커서 워킹셋이 캐시에 들어가지 않음 -> 트래픽이 DRAM까지 내려감

### 그래프 읽는 법

<figure>
  <img src="/assets/images/roofline-model.png" alt="Roofline model for AMD Opteron X2 and Opteron X2 vs. X4">
  <figcaption>Figure 1. Roofline model for (a) AMD Opteron X2 and (b) Opteron X2 vs. X4 (Williams et al., 2009)</figcaption>
</figure>

  - floating-point performance, operational intensity, memory performance를 2D graph에 표현한 것
    - x축: Operational Intensity (Flop/Byte) / y축: Attainable Performance (GFlop/s)
    - **두 축 모두 log scale (log-log 그래프)**
  - 2개의 line
    - horizontal line (peak floating-point performance)
      - HW spec이나 microbenchmarks를 통해 확인할 수 있음
      - HW spec에 의한 고정값이며 어떤 kernel도 이 line보다 높은 성능을 낼 수 없음
    - unit slope line (peak memory bandwidth)
      - memory system behind the caches(메모리 컨트롤러 + DRAM 채널 + DIMM)에 의해 정의됨
      - 그 컴퓨터의 메모리가 steady-state에서 낼 수 있는 최대 대역폭이며, HW에 의해 정해진 고정값
      - 기울기가 1인 이유: `y = BW * x`를 log-log에 그리면 `log y = log BW + log x`
  - `Attainable GFlops/sec = min(Peak Floating-Point Performance, Peak Memory Bandwidth * Operational Intensity)`

### Ridge Point가 의미하는 것
  - 두 line이 만나는 교차점으로, 그 컴퓨터의 전반적인 성능 특성을 한눈에 보여줌
  - x좌표 = `Peak Floating-Point Performance / Peak Memory Bandwidth` (그 머신의 Flop/Byte 균형점)
  - **커널** 관점: ridge point 기준 왼쪽에 위치하면 memory-bound, 오른쪽에 위치하면 compute-bound
  - **머신** 관점: ridge point의 위치 자체가 그 컴퓨터의 성격을 말해줌

    | ridge point 위치 | 의미 |
    | ------------- | ------------- |
    | 왼쪽 (작은 OI) | 대역폭에 여유가 있는 균형 잡힌 머신. 웬만한 커널도 peak에 도달하기 쉬움 |
    | 오른쪽 (큰 OI) | 연산 대비 대역폭이 빈약. 아주 높은 intensity의 커널만 peak에 도달 |


## Ceiling: 무엇을 먼저 고칠 것인가
### 지붕 아래의 천장들

### 천장을 쌓는 순서 = 최적화 순서

### 커널의 위치에 따라 갈리는 처방


## 3Cs와 Operational Intensity: 점을 오른쪽으로 밀기


## References
- [Roofline: An Insightful Visual Performance Model for Multicore Architectures (Williams et al., 2009)](https://dl.acm.org/doi/10.1145/1498765.1498785)
- [Hitting the Memory Wall: Implications of the Obvious (Wulf & McKee, 1995)](https://dl.acm.org/doi/10.1145/216585.216588)
- [Roofline Model 정리 (블로그)](https://tech-sherpa.tistory.com/17)
