---
layout: post
title:  "v8!, 넌 도대체 뭐니?"
author: "SeungYong Lee"
categories: [principle]
tags: [v8]
image: assets/images/Who-are-you!.gif
---
독자 여러분, 안녕하세요!

최근 Google의 ASAN(Address Sanitizer)에 관한 심도 있는 기술 문서를 접한 이후, 브라우저 보안의 구조적 취약점과 관련한 학문적 관심이 한층 더 깊어졌습니다. 이 글에서는 자바스크립트 및 웹 어셈블리 엔진인 V8에 대한 기술적 배경과 그 구조를 정리하고, 향후 V8 기반의 1-day 취약점 분석을 위한 사전 이해로 활용될 수 있도록 내용을 체계적으로 서술하고자 합니다.

본격적인 기술 분석에 앞서, "V8"이라는 명칭 자체가 가지는 기원에 대해 흥미를 가졌고, 이에 대한 소소한 고찰로 글을 시작하려 합니다.일반적으로 "V8"이라는 단어는 자동차 공학 분야에서 8기통 엔진을 지칭하는 용어로 널리 사용됩니다. V형으로 배열된 8개의 실린더가 피스톤의 왕복 운동을 통해 강력한 동력을 생성하는 이 엔진은, 4000cc 이상의 고배기량을 바탕으로 탁월한 퍼포먼스를 자랑하기때문에 머슬카나 고성능 스포츠카, 대형 SUV 등에서 널리 채택되고 있으며, 연료 효율보다는 출력 성능이 강조되어 "기름 먹는 하마" 라는 별칭을 가지고 있기도합니다.

"8기통은 미국이 태생이다" 라고 많은 사람들이 오해를 하고 있지만, V8 엔진이 미국에서 처음 고안된 것은 아닙니다. 자원이 풍부한 미국은 8기통을 세계 최초로 양산을 시작한 나라일 뿐, 기술적 기원은 1901년 프랑스의 한 공학자 Léon Levavasseur로 거슬러올라갑니다. 그는 항공기에 사용할 엔진을 설계하면서 직렬 4기통의 기통 수를 6개로 늘리려는 기술적 한계를 극복하기 위해 많은 시행착오를 겪었습니다. 하지만 그가 고안해낸 방법으로는 크랭크 축이 길어져 엔진이 원활하게 작동 하지 않자 그는 깊은 고민에 빠져 고뇌하였습니다. 

시간이 흐른 뒤, 그는 직렬 4기통 2개를 결합하는 방법을 고안해내어 합쳐보는 시도를 하였고 이후 원활하게 동작하는 것을 확인하며, 현대적인 의미의 V8 구조가 처음으로 세상에 등장하게 되었습니다. 이처럼 강력한 힘을 자랑하며 기술적 완성도를 상징하는 8기통 엔진에서 영감을 받아 Google은 자사의 차세대 자바스크립트엔진 프로젝트에 "V8"이라는 명칭을 부여했습니다.

2006년 가을 "Lars Bak"이 Google과의 인연을 시작함으로써 그를 중심으로 한 소수의 엔지니어들이 덴마크의 한 농가 별채에서 V8 개발을 수행했습니다.

V8은 개발 중에는 Google의 내부 비밀 프로젝트로써 비공개 CVS(Concurrent Versions System) 저장소에서 개발되었습니다. 하지만 2008년 9월 2일, Google Chrome의 공식 출시와 함께 V8 역시 오픈소스로 전환되어 전 세계 개발자들에게 공개되었습니다. 기술적으로 V8은 자바스크립트 코드의 실행 속도를 극대화하는 데 초점을 맞춰 C++ 로 개발되었으며, JIT(Just In Time) 컴파일러를 채택했습니다. 

JIT 덕분에 실행 시점에 자바스크립트 코드를 기계어로 변환함으로써 성능을 극적으로 향상시킬 수 있는 구조적 장점을 가졌지만 C/C++계열 특성상 Coder가 메모리를 수동으로 관리해줘야 하기 때문에 해커들은 그들의 실수의 틈을 찾아내고 이로 인해 발생하는 Memory Curruption은 종종 보안 취약점으로 이어졌습니다. 실제로 미국 백악관에서는 메모리 안정성 이슈를 해소하기 위해 Rust와 같은 메모리 안전 언어 도입을 권고하기도 했습니다.

V8의 실행 구조는 다음과 같은 다단계의 JIT 실행 파이프라인으로 갖추고 있습니다.

<div align="center" style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; margin: 0;">
[ Parser ]
↓
[ Ignition Interpreter ]
↓
[ Sparkplug ]
↓
[ Maglev ]
↓
[ TurboFan ]
</pre>
</div>

1. 파서 (Parser)<br>
자바스크립트 소스를 어휘 분석과 파싱을 통해 토큰화하고 AST(Abstract Syntax Tree)로 변환하는데, 이는 코드 구조를 표현하는 IR(Intermediate Representation)으로서 이후 단계의 기반이 됩니다.

2. 바이트코드 생성 (Ignition Interpreter)<br>
V8의 인터프리터인 Ignition은 AST를 입력으로 바이트코드를 생성하고 실행합니다. 이 단계에서 레지스터 기반 가상 머신으로 동작하며, 생성된 바이트코드를 한 단계씩 해석하여 실행하는데, 실행 성능은 느리지만, 초기 실행 및 메모리 효율 면에서 유리하며 애플리케이션의 빠른 시작에 기여합니다. 또한, 타입 피드백 수집을 위해 각 바이트코드 연산 실행 시 실행 중인 데이터 타입 정보를 피드백 벡터라는 자료구조에 기록합니다.

3. 베이스라인 JIT 컴파일 (Sparkplug)<br>
Ignition Interpreter으로 어느 정도 실행된 코드가 있다면, V8은 Sparkplug라 불리는 비최적화 컴파일러를 통해 바이트코드를 네이티브 머신 코드로 직접 컴파일합니다. Sparkplug는 V8 v9.1(Chrome 91)에서 도입된 신규 컴파일 단계로, Ignition Interpreter와 최적화 컴파일러들(Maglev, Turbofan) 사이에 위치합니다. Sparkplug는 빠른 컴파일 속도를 최우선 목표로 설계되어, TurboFan보다 적극적으로 잦은 컴파일이 가능하며 작은 함수도 신속히 기계어로 전환 할 수 있습니다. 이 단계에서 생성되는 코드는 최적화가 거의 적용되지 않았지만 인터프리터 바이트코드 실행보다 훨씬 빠르기 때문에, 애플리케이션의 실제 체감 성능을 5–15% 향상시키는 효과를 가져왔습니다.

4. 중간 최적화 JIT 컴파일 (Maglev)<br>
2023년부터 V8의 JIT 파이프라인에 도입된 Maglev(MAGnetized Low-latency Execution Velocity)는 Sparkplug와 TurboFan 사이의 성능 간극을 효과적으로 메워주는 중간 최적화 JIT 컴파일러입니다. 이 단계에서 SSA(Static Single Assignment) 기반의 선형 블록형 IR을 사용하며, 레지스터 기반 백엔드로 동작하여 Sparkplug보다 높은 최적화 성능을 제공하면서도 TurboFan보다 빠른 컴파일 속도를 자랑합니다. 전체적으로 단일 패스 방식의 빠른 코드 생성을 유지하면서도, 조건 분기, 호출 인라이닝, 피연산자 타입 추론 등 선택적인 최적화 기법을 적용하여 반복 루프와 같이 짧지만 빈번히 실행되는 함수에 특화된 JIT 처리를 제공합니다.
또한, Ignition Interpreter 실행 중 hot loop가 감지되면 On-Stack Replacement(OSR)를 통해 실행 중인 스택 프레임을 동적으로 JIT 최적화된 코드로 교체할 수 있으며, 이는 브라우저의 응답성을 떨어뜨리지 않으면서도 성능을 확보하는 데 매우 큰 역활을 합니다.

5. 고급 최적화 JIT 컴파일 (TurboFan)<br>
TurboFan은 V8 엔진의 마지막 JIT 계층이자 가장 정교한 최적화 컴파일러로, 일정 횟수 이상 호출되어 “핫스팟(hot)”으로 분류된 함수에 대해 실행됩니다. 이 단계에서는 Ignition과 초기 JIT 단계에서 수집된 타입 피드백(feedback vector)을 바탕으로 추측 기반(speculative) 최적화를 적용합니다. 그러면서 SSA 기반의 Sea‑of‑Nodes IR 또는 최근에는 더 향상된 Turboshaft(IR) 구조를 활용해 여러 최적화 패스를 수행합니다. 이 패스는 함수 인라이닝, 상수 전파, 범위 분석(range analysis), 경계 검사 제거, 중복 연산 제거, 히든 클래스 접근 최적화, 인라인 캐싱 적용 등을 포함합니다. TurboFan이 생성한 머신 코드는 가드 체크(guard checks)를 포함해 잘못된 타입이나 예상치 못한 상황이 발생할 경우 자동으로 비최적화 상태로 전환돼 Ignition Interpreter나 이전 JIT 단계의 코드로 복귀합니다.

요약하면, 현재의 V8은 Parser → Ignition Interpreter → Sparkplug(비최적화 JIT) → Maglev(중간 최적화 JIT) → TurboFan(고급 최적화 JIT)의 다단계 파이프라인을 통해 초기 구동 시간과 런타임 성능 간 균형을 이루는 구조를 취하고 있습니다.

즉, V8의 JIT 컴파일러는 기존 인터프리터 방식의 단점인 반복된 명령어 실행 시 매번 동일한 해석이 필요한 비효율성을 보완하여, 실시간으로 컴파일된 기계어를 재활용함으로써 구조적 성능을 비약적으로 향상시킵니다. 이러한 다단계 실행 파이프라인은 자바스크립트 실행 환경의 실질적인 속도 개선뿐만 아니라, 현대 웹 애플리케이션에서 요구하는 높은 반응성과 복잡도 처리를 가능하게 만든 핵심 요소라고 생각합니다.

끝으로, 이 글이 브라우저 내부 구조와 V8 JIT 파이프라인에 대한 기술적 이해를 넓히는 데 작은 보탬이 되었기를 바랍니다.
다음 글에서는 TurboFan 단계에서 발생한 1-day 취약점인 CVE-2023-2033의 원인과 공격 벡터에 대해 분석해보고자 합니다.

저의 글 읽어주셔서 진심으로 감사드립니다.

### [ Reference ]
- https://blog.bitsrc.io/secret-behind-javascript-performance-v8-hidden-classes-ba4d0ebfb89d
- https://ko.wikipedia.org/wiki/V8_엔진
- https://oldmachinepress.com/2016/05/28/antoinette-levavasseur-aircraft-engines/
- https://rnfltpgus.github.io/knowledge/v8-engine/
- https://jaehyeon48.github.io/javascript/google-v8-engine/
- https://v8.dev/blog/sparkplug
- https://v8.dev/blog/maglev
- https://github.blog/security/vulnerability-research/getting-rce-in-chrome-with-incomplete-object-initialization-in-the-maglev-compiler/