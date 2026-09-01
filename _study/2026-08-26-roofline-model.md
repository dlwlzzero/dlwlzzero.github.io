---
title: "Roofline: An Insightful Visual Performance Model for Multicore Architectures"
type: review
date: 2026-08-26
last_modified_at: 2026-08-27
---

<h2 id="s1">Background</h2>

<h3>Recent switch to multicore</h3>
<p style="margin:0 0 8px; color:var(--fg); font-size:17px; line-height:1.75">ISA(Instruction Set Architecture, 명령어 집합 구조)의 다양화</p>
<table style="margin:16px 0 8px">
  <thead><tr>
    <th><span lang="ko">항목</span></th>
    <th>CISC = Complex Instruction Set Computer</th>
    <th>RISC = Reduced Instruction Set Computer</th>
  </tr></thead>
  <tbody>
    <tr>
      <td style="min-width:120px">명령어 길이</td>
      <td style="min-width:220px">가변 길이 명령어</td>
      <td style="min-width:220px">고정 길이 명령어</td>
    </tr>
    <tr>
      <td>파이프라이닝</td>
      <td>pipelining <strong style="font-weight:600; color:var(--fg)">어려움</strong></td>
      <td>pipelining <strong style="font-weight:600; color:var(--fg)">쉬움</strong></td>
    </tr>
    <tr>
      <td>메모리 접근</td>
      <td>메모리 오퍼랜드 직접 사용<br>(연산 명령어가 메모리를 직접 읽고 씀)</td>
      <td>Load/Store 구조<br>(메모리 접근은 <code>lw/sw</code>만, 나머지 연산은 레지스터끼리만)</td>
    </tr>
    <tr>
      <td>명령어 해석</td>
      <td>마이크로코드 있음<br>(복잡한 명령어를 칩 내부의 작은 프로그램이 해석해서 실행)</td>
      <td>마이크로코드 없음<br>(hardwired 제어)</td>
    </tr>
    <tr>
      <td>복잡도의 위치</td>
      <td>복잡도가 <strong style="font-weight:600; color:var(--fg)">하드웨어</strong>에 있음</td>
      <td>복잡도가 <strong style="font-weight:600; color:var(--fg)">컴파일러</strong>에 있음</td>
    </tr>
    <tr>
      <td>코드 예시</td>
      <td><code>ADD [eax + ebx*4], 5</code></td>
      <td><pre style="margin:0; font-family:var(--mono); font-size:12.5px; line-height:1.7; padding:10px 12px; overflow-x:auto; background:var(--bg); border:1px solid var(--rule); border-radius:4px; color:var(--fg)">sll   $t0, $s1, 2      # ebx*4
add   $t0, $t0, $s0    # + eax → 주소
lw    $t1, 0($t0)      # 메모리 → 레지스터
addi  $t1, $t1, 5      # 레지스터에서 연산
sw    $t1, 0($t0)      # 레지스터 → 메모리</pre></td>
    </tr>
    <tr>
      <td>대표 ISA</td>
      <td>x86, VAX, 68000</td>
      <td>MIPS, SPARC, ARM, RISC-V</td>
    </tr>
  </tbody>
</table>

<p style="margin:28px 0 8px; color:var(--fg); font-size:17px; line-height:1.75">Cache vs Local Memory</p>
<table style="margin:16px 0 8px">
  <thead><tr>
    <th><span lang="ko">항목</span></th>
    <th>Cache</th>
    <th>Local store / Scratchpad</th>
  </tr></thead>
  <tbody>
    <tr>
      <td style="min-width:130px">데이터 이동 주체</td>
      <td style="min-width:220px">하드웨어가 자동 (miss 시 fill, coherence 전부 처리)</td>
      <td style="min-width:220px">소프트웨어가 명시적으로 DMA/버퍼 관리</td>
    </tr>
    <tr>
      <td>주소 공간</td>
      <td>메인 메모리와 동일</td>
      <td>별도의 독립 주소 공간</td>
    </tr>
    <tr>
      <td>하드웨어 비용</td>
      <td>SRAM + 태그 배열 + 비교기 + 교체 로직 + coherence (비쌈)</td>
      <td>SRAM Only (저렴)</td>
    </tr>
    <tr>
      <td>접근 지연</td>
      <td>예측 불가 (hit/miss에 따라 가변)</td>
      <td>항상 고정 → 사이클 단위 스케줄링 가능</td>
    </tr>
    <tr>
      <td>적합한 워크로드</td>
      <td>접근 패턴이 불규칙/동적인 범용 코드</td>
      <td>컴파일 타임에 패턴을 아는 규칙적 연산</td>
    </tr>
    <tr>
      <td>대표 사례</td>
      <td>범용 CPU의 L1/L2/L3 Cache</td>
      <td>CUDA shared memory, Hexagon VTCM</td>
    </tr>
  </tbody>
</table>

<h3>Memory Wall: 왜 메모리가 병목인가</h3>
<p style="margin:0 0 8px; color:var(--fg); font-size:17px; line-height:1.75">연산 성능과 메모리 성능의 개선 속도가 수십 년째 벌어져 옴</p>
<table style="margin:16px 0 20px">
  <thead><tr>
    <th><span lang="ko">항목</span></th>
    <th><span lang="ko">연간 개선률 (경험적 추세)</span></th>
    <th><span lang="ko">결과</span></th>
  </tr></thead>
  <tbody>
    <tr>
      <td>프로세서 연산 성능</td>
      <td style="min-width:220px">약 50%/년 (2000년대 중반 이후엔 코어 수 증가로 지속)</td>
      <td style="min-width:200px">매년 Flop/s는 계속 늘어남</td>
    </tr>
    <tr>
      <td>Off-chip DRAM 대역폭</td>
      <td>약 25%/년</td>
      <td>연산 대비 상대적으로 계속 뒤처짐</td>
    </tr>
    <tr>
      <td>Off-chip DRAM 지연(latency)</td>
      <td>약 7%/년</td>
      <td>격차가 가장 큼. prefetch/멀티스레딩으로 숨겨야 함</td>
    </tr>
  </tbody>
</table>
<div>
  <p style="margin:0; color:var(--fg); font-size:17px; line-height:1.75">그 결과, 칩이 낼 수 있는 Byte/Flop 비율이 해마다 낮아짐</p>
  <div style="margin:8px 0 0; padding-left:18px; border-left:1px solid var(--rule); display:flex; flex-direction:column; gap:6px">
    <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">같은 코드라도 시간이 지날수록 연산기가 아니라 메모리에서 굶주리게 됨</p>
    <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">멀티코어는 코어 수만 늘릴 뿐 off-chip 핀 대역폭은 공유 → 코어당 대역폭은 오히려 감소</p>
  </div>
</div>

<h3>Insight over Accuracy: The 3Cs Model for Caches</h3>
<div style="display:flex; flex-direction:column; gap:8px">
  <p style="margin:0; color:var(--fg); font-size:17px; line-height:1.75">3Cs model for caches <code>Total Cache Miss = Compulsory + Capacity + Conflict</code></p>
  <p style="margin:0; padding-left:18px; border-left:1px solid var(--rule); color:var(--fg-2); font-size:16px; line-height:1.72">Cache miss가 왜 발생하는지를 3가지 원인으로 분해하는 모델 (by Mark Hill)</p>
</div>
<table style="margin:16px 0 20px">
  <thead><tr>
    <th><span lang="ko">항목</span></th>
    <th>Compulsory (cold miss)</th>
    <th>Capacity</th>
    <th>Conflict</th>
  </tr></thead>
  <tbody>
    <tr>
      <td style="min-width:110px">정의</td>
      <td style="min-width:230px">그 블록에 처음 접근할 때, 캐시가 아무리 커도 데이터는 한 번은 메모리에서 올라와야 하므로 피할 수 없음</td>
      <td style="min-width:230px">프로그램이 그 구간에서 필요로 하는 워킹셋이 캐시 용량보다 커서, 한 번 올라왔던 블록이 쫓겨났다가 다시 필요해지는 경우</td>
      <td style="min-width:230px">용량은 남는데 같은 set이 매핑되는 블록이 associativity보다 많아서 쫓겨나는 경우. Fully-associative였다면 안 났을 miss</td>
    </tr>
    <tr>
      <td>측정 방법</td>
      <td>무한 크기 fully-associative 캐시의 미스 수</td>
      <td>(크기 C의 fully-associative 미스) − Compulsory</td>
      <td>(실제 캐시 미스) − (같은 크기 fully-associative 미스)</td>
    </tr>
    <tr>
      <td>대안책</td>
      <td>블록 크기 증가, HW/SW prefetch</td>
      <td>캐시 용량 증가, tiling/blocking으로 워킹셋 축소</td>
      <td>associativity 증가, victim cache, 배열 padding</td>
    </tr>
  </tbody>
</table>
<div>
  <p style="margin:0; color:var(--fg); font-size:17px; line-height:1.75">What the 3Cs model ignores</p>
  <p style="margin:8px 0 0; padding-left:18px; border-left:1px solid var(--rule); color:var(--fg-2); font-size:16px; line-height:1.72">block size, block-allocation/replacement policy 등을 무시함 → 용량과 associativity만 고려함</p>
</div>

<h3>Performance Models</h3>
<div style="display:flex; flex-direction:column; gap:10px">
  <p style="margin:0; color:var(--fg); font-size:17px; line-height:1.75">프로그램의 성능을 예측하는 모델: stochastic analytical model, statistical performance model</p>
  <p style="margin:0; color:var(--fg); font-size:17px; line-height:1.75"><strong style="font-weight:600">bound and bottleneck analysis</strong>: Amdahl's Law (병렬 컴퓨터의 성능 향상은 병렬 프로그램의 직렬(순차) 부분에 의해 제한됨)</p>
</div>

<h2 id="s2">Roofline Model <span lang="ko">뜯어보기</span></h2>

<h3>한 줄 요약</h3>
<div style="display:flex; flex-direction:column; gap:8px">
  <p style="margin:0; color:var(--fg); font-size:16.5px; line-height:1.72">주어진 컴퓨팅 환경에서 성능의 제약조건이 어느 부분인지 쉽게 시각화할 수 있는 도구</p>
  <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">컴퓨팅 성능은 주로 ①메모리 대역폭과 ②연산기(최대 연산 능력)에 영향을 받으며, Roofline model에서는 이를 하나의 그래프에 시각화</p>
</div>

<h3>Operational Intensity</h3>
<div style="display:flex; flex-direction:column; gap:0">
  <div style="border-top:1px solid var(--rule); padding:16px 0">
    <p style="margin:0 0 8px; font-size:16.5px; font-weight:600; color:var(--fg)">정의</p>
    <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72"><code>Flops / Byte</code>, DRAM에서 가져온 바이트당 몇 번의 부동소수점 연산을 하는지를 의미</p>
    <p style="margin:8px 0 0; padding-left:18px; border-left:1px solid var(--rule); color:var(--fg-2); font-size:15.5px; line-height:1.7">초당 연산량(Flop/s)이 아니라 <strong style="font-weight:600; color:var(--fg)">비율(ratio)</strong>임에 주의</p>
  </div>
  <div style="border-top:1px solid var(--rule); padding:16px 0">
    <p style="margin:0 0 8px; font-size:16.5px; font-weight:600; color:var(--fg)">측정 방법</p>
    <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">caches와 메모리 사이의 traffic을 측정하면 됨</p>
  </div>
  <div style="border-top:1px solid var(--rule); border-bottom:1px solid var(--rule); padding:16px 0">
    <p style="margin:0 0 8px; font-size:16.5px; font-weight:600; color:var(--fg)">전제</p>
    <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">커널이 반복 사용하는 데이터가 온칩 캐시 용량보다 커서 워킹셋이 캐시에 들어가지 않음 → 트래픽이 DRAM까지 내려감</p>
  </div>
</div>

<h3>그래프 읽는 법</h3>
<figure style="margin:1.6em 0">
  <img src="/assets/images/roofline-model.png" alt="Roofline model for AMD Opteron X2 and Opteron X2 vs. X4" style="display:block; margin:0 auto">
  <figcaption style="margin-top:10px; text-align:center; font-family:var(--mono); font-size:12.5px; line-height:1.6; color:var(--fg-3)">Roofline model — Opteron X2, X2 vs. X4</figcaption>
</figure>
<div>
  <p style="margin:0; color:var(--fg); font-size:17px; line-height:1.75">floating-point performance, operational intensity, memory performance 세 가지를 하나의 2D graph에 담은 것</p>
  <div style="margin:8px 0 0; padding-left:18px; border-left:1px solid var(--rule); display:flex; flex-direction:column; gap:6px">
    <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">x축: Operational Intensity (Flop/Byte) / y축: Attainable Performance (GFlop/s)</p>
    <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72"><strong style="font-weight:600; color:var(--fg)">두 축 모두 log scale (log-log 그래프)</strong></p>
  </div>
</div>
<p style="margin:20px 0 12px; font-family:var(--mono); font-size:12px; letter-spacing:0.06em; text-transform:uppercase; color:var(--fg-3)"><span lang="ko">2개의</span> line</p>
<div style="display:grid; gap:14px">
  <div>
    <p style="margin:0 0 10px; font-size:16.5px; font-weight:600; color:var(--fg)">horizontal line — peak floating-point performance</p>
    <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">HW spec이나 microbenchmarks로 확인할 수 있는 고정값이며, 어떤 kernel도 이 line보다 높은 성능을 낼 수 없음</p>
  </div>
  <div>
    <p style="margin:0 0 10px; font-size:16.5px; font-weight:600; color:var(--fg)">unit slope line — peak memory bandwidth</p>
    <div style="display:flex; flex-direction:column; gap:8px">
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">캐시 뒤쪽의 메모리 시스템(메모리 컨트롤러 + DRAM 채널 + DIMM)이 결정함</p>
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">그 컴퓨터의 메모리가 steady-state에서 낼 수 있는 최대 대역폭이며, HW에 의해 정해진 고정값</p>
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">기울기가 1인 이유: <code>y = BW * x</code>를 log-log에 그리면 <code>log y = log BW + log x</code></p>
    </div>
  </div>
</div>
<p style="margin:16px 0 0; padding:10px 14px; background:var(--surface); border:1px solid var(--rule); border-radius:4px; font-family:var(--mono); font-size:14px; color:var(--fg)">Attainable GFlops/sec = min(Peak Floating-Point Performance, Peak Memory Bandwidth * Operational Intensity)</p>

<h3>Ridge Point가 의미하는 것</h3>
<div style="display:flex; flex-direction:column; gap:10px">
  <p style="margin:0; color:var(--fg); font-size:17px; line-height:1.75">두 line이 만나는 교차점으로, 그 컴퓨터의 전반적인 성능 특성을 한눈에 보여줌</p>
  <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">x좌표 = <code>Peak Floating-Point Performance / Peak Memory Bandwidth</code> (그 머신의 Flop/Byte 균형점)</p>
  <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72"><strong style="font-weight:600; color:var(--fg)">커널</strong> 관점: ridge point 기준 왼쪽에 위치하면 memory-bound, 오른쪽에 위치하면 compute-bound</p>
  <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72"><strong style="font-weight:600; color:var(--fg)">머신</strong> 관점: ridge point의 위치 자체가 그 컴퓨터의 성격을 말해줌</p>
</div>
<table style="margin:20px 0 8px">
  <thead><tr>
    <th>ridge point <span lang="ko">위치</span></th>
    <th><span lang="ko">의미</span></th>
  </tr></thead>
  <tbody>
    <tr>
      <td>왼쪽 (작은 OI)</td>
      <td>대역폭에 여유가 있는 균형 잡힌 머신. 웬만한 커널도 peak에 도달하기 쉬움</td>
    </tr>
    <tr>
      <td>오른쪽 (큰 OI)</td>
      <td>연산 대비 대역폭이 빈약. 아주 높은 intensity의 커널만 peak에 도달</td>
    </tr>
  </tbody>
</table>

<h2 id="s3">Adding Ceilings to the Model</h2>
<figure style="margin:0 0 1.6em">
  <img src="/assets/images/roofline-model-2.png" alt="Roofline model with ceilings for Opteron X2" style="display:block; margin:0 auto">
  <figcaption style="margin-top:10px; text-align:center; font-family:var(--mono); font-size:12.5px; line-height:1.6; color:var(--fg-3)">Roofline model with ceilings — Opteron X2</figcaption>
</figure>

<h3>Add Multiple Ceilings (어떤 Optimization부터 적용할 것인가)</h3>
<p style="margin:0 0 16px; color:var(--fg); font-size:17px; line-height:1.75">ceiling의 역할: 대응하는 optimization을 적용하기 전까지는 그 위의 performance line에 도달할 수 없음</p>
<div style="display:grid; gap:14px">
  <div>
    <p style="margin:0 0 10px; font-size:16.5px; font-weight:600; color:var(--fg)">Computational (Performance) Ceiling</p>
    <div style="display:flex; flex-direction:column; gap:8px">
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72"><span style="color:var(--fg)">ILP(instruction-level parallelism) 향상:</span> superscalar 프로세서가 매 클럭 최대한 많은 명령어를 fetch/execute/commit 하도록 loop unrolling 등을 적용</p>
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72"><span style="color:var(--fg)">SIMD 적용:</span> 인접한 여러 operand를 묶어 한 번에 연산</p>
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72"><span style="color:var(--fg)">floating-point 연산의 균형:</span> 덧셈과 곱셈의 비중이 동일해야 함 (FP adder와 multiplier가 별도 유닛이라 1:1일 때만 둘 다 포화됨)</p>
    </div>
  </div>
  <div>
    <p style="margin:0 0 10px; font-size:16.5px; font-weight:600; color:var(--fg)">Bandwidth (Memory) Ceiling</p>
    <div style="display:flex; flex-direction:column; gap:8px">
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72"><span style="color:var(--fg)">unit stride 접근이 되도록 loop 재구성:</span> HW prefetcher가 접근 패턴을 인식할 수 있게 되어 prefetch 성능이 올라감</p>
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72"><span style="color:var(--fg)">memory affinity 확보:</span> 데이터와 그 데이터를 처리할 스레드를 동일한 메모리-프로세서 쌍에 배치하여, 프로세서가 다른 칩에 붙은 메모리에 거의 접근할 일이 없도록 만드는 것</p>
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">software prefetching</p>
    </div>
  </div>
</div>
<div style="display:flex; flex-direction:column; gap:0; margin-top:20px">
  <div style="border-top:1px solid var(--rule); padding:16px 0">
    <p style="margin:0 0 8px; font-size:16.5px; font-weight:600; color:var(--fg)">ceiling은 아래에서 위로, 구현 난이도 순으로 쌓임</p>
    <div style="display:flex; flex-direction:column; gap:6px; padding-left:18px; border-left:1px solid var(--rule)">
      <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">아래쪽 = 컴파일러나 런타임이 알아서 해주는 것 / 위쪽 = 프로그래머가 직접 손대야 하는 것</p>
      <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">아래 ceiling을 뚫지 못하면 그 위의 ceiling은 의미가 없음 → 아래부터 차례로 적용</p>
    </div>
  </div>
  <div style="border-top:1px solid var(--rule); border-bottom:1px solid var(--rule); padding:16px 0">
    <p style="margin:0 0 8px; font-size:16.5px; font-weight:600; color:var(--fg)">ceiling 사이의 간격은 그 optimization의 잠재 이득을 의미함</p>
    <div style="display:flex; flex-direction:column; gap:6px; padding-left:18px; border-left:1px solid var(--rule)">
      <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">간격이 넓다 → 적용했을 때 얻을 성능이 큼</p>
      <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">간격이 좁다 → 공들여 적용해도 얻을 게 별로 없음</p>
      <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">"무엇을 먼저 할 것인가"뿐 아니라 "할 가치가 있는가"까지 그래프 하나로 판단 가능</p>
    </div>
  </div>
</div>
<p style="margin:20px 0 8px; color:var(--fg); font-size:17px; line-height:1.75">커널의 OI가 ridge point 기준 어디에 있느냐에 따라 먼저 손댈 ceiling이 갈림</p>
<table style="margin:16px 0 8px">
  <thead><tr>
    <th><span lang="ko">커널 위치</span></th>
    <th><span lang="ko">먼저 뚫을</span> ceiling</th>
  </tr></thead>
  <tbody>
    <tr>
      <td>왼쪽 (memory-bound)</td>
      <td>Bandwidth ceiling (unit stride, memory affinity, prefetch)</td>
    </tr>
    <tr>
      <td>오른쪽 (compute-bound)</td>
      <td>Computational ceiling (ILP, SIMD, add/mul 균형)</td>
    </tr>
    <tr>
      <td>ridge point 근처</td>
      <td>양쪽 모두 필요할 수 있음</td>
    </tr>
  </tbody>
</table>

<h2 id="s4">Tying the 3Cs to Operational Intensity</h2>
<p style="margin:0 0 16px; color:var(--fg); font-size:17px; line-height:1.75">Operational Intensity가 고정되어 있지 않을 수 있음</p>
<div style="display:flex; flex-direction:column; gap:0">
  <div style="border-top:1px solid var(--rule); padding:16px 0">
    <p style="margin:0 0 10px; font-size:16.5px; font-weight:600; color:var(--fg)">일부 커널에서는 연산의 사이즈에 따라서 OI가 증가할 수도 있음</p>
    <p style="margin:0 0 10px; font-family:var(--mono); font-size:12px; letter-spacing:0.06em; text-transform:uppercase; color:var(--fg-3)"><span lang="ko">예제</span> — Dense Matrix Multiply (IN1=IN2=OUT=[N,N] size, single precision)</p>
    <div style="display:flex; flex-direction:column; gap:8px">
      <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7"><span style="color:var(--fg)">Flops:</span> <code>ADD_OP_CNT+MUL_OP_CNT=2*(N^3)</code></p>
      <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7"><span style="color:var(--fg)">Bytes:</span> <code>IN1_SIZE+IN2_SIZE+OUT_SIZE=3*(N^2)*(4 Bytes)= 12*(N^2)</code> (single precision 기준, double이면 <code>24*(N^2)</code>)</p>
      <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7"><span style="color:var(--fg)">Operational Intensity:</span> <code>2*(N^3) / (12*(N^2)) = N/6</code></p>
      <div>
        <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">단, <code>N/6</code>은 각 원소를 DRAM에서 딱 한 번만 읽을 때(= compulsory miss만 발생) 도달 가능한 <strong style="font-weight:600; color:var(--fg)">상한</strong></p>
        <div style="margin:8px 0 0; padding-left:18px; border-left:1px solid var(--rule); display:flex; flex-direction:column; gap:6px">
          <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">blocking 없이 짜면 IN2를 N번 다시 읽어 트래픽이 O(N^3)이 되고, OI는 N과 무관한 상수로 주저앉음</p>
          <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">즉 "문제 크기가 커지면 OI가 오른다"는 것 자체가 캐시를 잘 썼을 때의 이야기</p>
        </div>
      </div>
    </div>
  </div>
  <div style="border-top:1px solid var(--rule); border-bottom:1px solid var(--rule); padding:16px 0">
    <p style="margin:0 0 10px; font-size:16.5px; font-weight:600; color:var(--fg)">Cache로 인해 DRAM에 데이터를 access하는 양이 줄어들 수 있음</p>
    <div style="display:flex; flex-direction:column; gap:8px">
      <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">Cache performance를 높이는 것이 OI를 높이는 데도 도움이 됨</p>
      <div>
        <p style="margin:0; color:var(--fg-2); font-size:16px; line-height:1.72">3Cs model을 Roofline model과 이어서 볼 수 있음</p>
        <div style="margin:8px 0 0; padding-left:18px; border-left:1px solid var(--rule); display:flex; flex-direction:column; gap:6px">
          <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">Compulsory miss만 남았을 때가 DRAM 트래픽이 최소인 상태 → 그 커널이 가질 수 있는 가장 높은 OI</p>
          <p style="margin:0; color:var(--fg-2); font-size:15.5px; line-height:1.7">Conflict/Capacity miss는 불필요한 트래픽을 만들어 커널의 OI를 끌어내림</p>
        </div>
      </div>
    </div>
  </div>
</div>

<h2 id="s5">References</h2>
<ul style="list-style:none; margin:0; padding:0; display:flex; flex-direction:column; gap:10px">
  <li><a href="https://dl.acm.org/doi/10.1145/1498765.1498785" style="font-size:16px">Roofline: An Insightful Visual Performance Model for Multicore Architectures (Williams et al., 2009)</a></li>
  <li><a href="https://dl.acm.org/doi/10.1145/216585.216588" style="font-size:16px">Hitting the Memory Wall: Implications of the Obvious (Wulf &amp; McKee, 1995)</a></li>
  <li><a href="https://tech-sherpa.tistory.com/17" style="font-size:16px">Roofline Model 정리 (블로그)</a></li>
</ul>
