---
title: "[Review] Roofline: An Insightful Visual Performance Model for Multicore Architectures"
date: 2026-08-26
last_modified_at: 2026-08-26
---

## Roofline Model이란 무엇인가?
- 주어진 컴퓨팅 환경에서 성능의 제약조건이 어느 부분인지 쉽게 시각화할 수 있는 도구
- 컴퓨팅 성능은 주로 ①메모리 대역폭과 ②연산기(최대 연산 능력)에 영향을 받으며, Roofline model에서는 이를 하나의 그래프에 시각화

## Roofline Model이 주장될 때의 배경
- HW Architecture의 다양화
  - ISA(Instruction Set Architecture, 명령어 집합 구조)의 변화
    | CISC=Complex Instruction Set Computer | RISC=Reduced Instruction Set Computer |
    | ------------- | ------------- |
    | 가변 길이 명령어 | 고정 길이 명령어 (전부 32비트) |
    | pipelining **불가** | pipelining **가능** |
    | 메모리 오퍼랜드 직접 사용<br>(연산 명령어가 메모리를 직접 읽고 씀)| Load/Store 구조<br>(메모리 접근은 `lw/sw`만, 나머지 연산은 레지스터끼리만) |
    | 마이크로코드 있음<br>(복잡한 명령어를 칩 내부의 작은 프로그램이 해석해서 실행)| 마이크로코드 없음<br>(hardwired 제어)|
    | 복잡도가 **하드웨어**에 있음 | 복잡도가 **컴파일러**에 있음 | 
    | `ADD [eax + ebx*4], 5`| `sll   $t0, $s1, 2      # ebx*4`<br>`add   $t0, $t0, $s0    # + eax → 주소`<br>`lw    $t1, 0($t0)      # 메모리 → 레지스터`<br>`addi  $t1, $t1, 5      # 레지스터에서 연산`<br>`sw    $t1, 0($t0)      # 레지스터 → 메모리`|
    | x86,VAX,68000 | MIPS,SPARC,ARM,RISC-V|
  - Cache vs Local Memory
  - Multicore 구조의 등장
- 3Cs (compulsory, capacity, and conflict misses) model for caches


## References
- https://dl.acm.org/doi/10.1145/1498765.1498785
- https://tech-sherpa.tistory.com/17

