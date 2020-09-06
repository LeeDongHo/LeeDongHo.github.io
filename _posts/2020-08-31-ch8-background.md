---
title: "OS ch8 메모리관리_1_background"
categories:
    - OS
tags:
    - OS,Memory management
---

# CH8-1. background
메모리는 현대 컴퓨터 시스템에서 핵심 작업이다. 메모리는 큰 바이트(byte)형태의 배열로 구성되어 있고 각각 주소를 갖는다.   
CPU는 pc(program counter)의 값에 따라서 메모리에서 instruction을 가져옵니다. 이런 instruction은 특정 메모리 주소에서 추가적인 로드, 저장작업을 발생시킬 수 있습니다.

일반적인 instruction의 실행 과정은 아래와 같습니다.

1. instruction이 메모리에서 패치(fetch)됩니다.
2. 패치된 instruction은 Decode 후 메모리에서 피연산자(operand)를 패치해옵니다.
3. 그 후 피연산자에 의해 instruction이 실행되고 실행 결과가 다시 메모리에 저장됩니다.

메모리 유닛은 메모리의 연속됨만 알고 그 외(instruction counter, indexing, indirction, literal address, data / instruction의 생성 방법)은 모릅니다.

## Basic Hardware
> 기호 메모리 주소(symbolic memory address)를 물리 주소(physical address)로 binding 합니다.

메인 메모리와 레지스터는 CPU가 바로 접근할 수 있는 유일한 장치들이다. **기계 명령어(machine instructions)은 메모리 주소만 argument로 취하고** 디스크 주소는 취하지 않습니다. 따라서 실행되는 명령어나 데이터들은 모두 CPU가 직접 접근 가능한 곳에 위치해야하며 그렇지 않은 경우(data가 메모리에 존재하지 않는 경우)엔 CPU는 데이터를 메모리에 옮기는 작업을 수행합니다.

1장에서 CPU의 **cache**에 대해 알아봤다. CPU는 빠른 데이터 접근 속도뿐만 아니라 멀티 유저(프로세스) 환경에서 다른 유저(프로세스)에게 메모리를 침범하지 못하도록 막아야한다. 이 작업은 운영체재가 하지 않고 하드웨어가 담당해야한다. (운영체재는 CPU와 메모리 사이에 존재하는 것이 아니기 때문에 운영체재가 이를 담당할 경우 상당한 손실이 존재한다.)

각 프로세스마다 독립된 메모리 공간을 소유한다. 이는 멀티 프로세스 환경에서 다른 프로세스에 의헤 메모리 영역이 침범당하는 현상을 근본적으로 방어해준다.
이 방법은 CPU의 base, limit 레지스터를 이용해서 구현할 수 있다. 
* Base register : 가장 작은 합법적인 물리 메모리 공간(legal physical memory address)을 보유한다.
* Limit register : 메모리 사용 공간 범위를 명시한다.
<p align = "center">
<img src="./image/base_limit_register.png" alt="base and limit register" width="350">
</p>
위 그림에서 base 레지스터는 300040의 물리 주소 값을 갖는다. limit는 120900이라는 값을 갖는데, 이는 [base, base + limit) - base 이상, base + limit 미만 사이의 메모리 공간을 프로세스가 갖는다는 의미이다.
즉 프로세스는 300040 ~ 420939의 공간을 독점적으로 소유한다.   
<br>
<br>
<p align = "center">
<img src="./image/protection_with_base_limit_register.png" alt="protection_with_base_limit_register" width="550">
</p>

그림 8.2를 통해 이 두 레지스터가 어떻게 다른 프로세스의 메모리 침범을 막는지를 보여준다.    
   
base, limit 레지스터는 오직 운영체제의 특권 명령(privileged instruction)만 로드할 수 있다. 이 명령은 커널 모드에서만 실행되어 사용자는 이 값을 수정할 수 없다.   
   
운영체제는 커널 영역에서 실행되며 운영체재의 메모리 공간과 사용자의 메모리 공간에 제약없이 접근할 수 있다. 이 특성 덕분에 운영체재가 사용자 메모리를 로드, 오류 발생 시 덤프(Dump)를 기록하거나, 시스템 콜(system call)의 매개변수를 수정하는 작업, 사용자 메모리 공간에서 I/O작업 실행 등을 할 수 있다.
또 Context-switch에서 실행중인 작업의 레지스터를 메인 메모리에 저장하는 것도 가능하게 해준다.


## Address Binding
일반적으로 프로그램은 디스크에 실행 가능한 이진파일 형태로 저장되어 있다. 프로그램이 실행될 때 디스크에서 메모리로 로드되며 실행중인 프로그램은 **프로세스**라고 부른다.   
메모리 관리에 의해 프로세스는 디스크와 메모리를 왔다갔다 할 수 있다. 디스크에서 메모리로 로드되기를 기다리는 프로세스들은 **input queue가 관리**한다.   
   
대부분의 시스템들은 유저 프로세스마다 물리 메모리를 부분적으로 할당해준다. 프로세스 메모리의 시작주소가 00000라고 해도 실제 물리 메모리는 00000부터 시작하지 않는다. 이 방법은 뒤에 기술한다.

**[사용자 프로그램의 실행 과정]**   <br><br>
<p align = "center">
<img src="./image/multistep_processing_user_program.png" alt="multistep_processing_user_program" width="350">
</p>

이 단계에서 주소는 다른 방식으로 표기될 수 있다.   
1. 소스 프로그램 (source file)에서 주소는 symbolic (int count 같은 변수)이다.
2. 컴파일러가 symbolic 주소를 상대적(relocatable) 주소로 **바인드**(시작 주소부터 14byte같은 방법)한다.
3. 링커(Linkage deitor or loader)가 상대적 주소를 물리적 주소로 **바인드(Bind)**한다.

메모리가 바인딩되는 시점에 따라 바인딩 기법은 3가지로 분류한다.
* Compile time : 컴파일 시점에 필요한 메모리 위치를 아는 경우 절대 코드(absolute code) 생성
    - 예를 들어 새로운 프로세스의 시작 위치(starting location)가 300400인 경우
    - MS-DOS의 .COM 포맷 프로그램이 이에 속한다.
* Load time : compile time에 메모리의 시작 위치를 모르는 경우 컴파일러는 상대적 위치를 생성하고 바인딩을 load time까지 미룬다. 만약 시작 주소가 바뀐 경우 변경된 값을 포함한 유저 코드만 다시 로드(reload) 한다.
* Execution time : 만약 프로세스가 실행중에 메모리 세그먼트를 다른 위치로 변경하는 경우 바인딩은 run time까지 미뤄진다. 이 작업은 하드웨어의 지원이 필요하며 현대의 컴퓨터 시스템은 대부분 이 바인딩 기법을 사용한다.

## Logical versus Physical Address Space
> * Logical address (논리 주소) : CPU에 의해 생성되는 주소
> * Physical address (물리 주소) : 메모리 장치가 취급하는 주소 (==메모리 장치의 메모리 주소 레지스터 (Memory-address register)에 로드되는 주소)
   
Compile time binding, Load time binding에서 주소 생성 방법은 논리 주소와 물리 주소가 동일하다. 하지만 Execution time binding에서 논리 주소와 물리 주소는 별개의 주소 공간이다. 즉 프로그램이 논리 주소 공간을 생성시킬 때 물리 주소 공간도 생성이 되고 **두 공간은 다른 공간**이다.   
논리 주소(logical address)는 가상 주소(virtual address)라고도 부른다.   

   
런타임 환경에서 가상 주소를 물리 주소로 맵핑(mapping)하는 것은 하드웨어의 MMU(Memory-management unit)가 담당한다. 맵핑 하는 방법에는 다양한 방법이 있는데 이는 나중에 설명한다.

간단한 MMU (simple MMU)는 우리가 앞서 살펴봤던 base, limit register를 이용한 방식이다. 이때 base register를 **relocation register**이라고도 부른다.
사용자 프로그램은 가상 주소만을 다루며 물리 주소에 직접 접근할 수 없다. 또 가상 주소 공간들은 **실제로 메모리 공간을 참조할 때 실제 위치가 할당**된다.   
**가상 주소가 여러 물리 주소에 바인딩 되는 개념은 메모리 관리 기법의 핵심이다.**
**가상 주소는 사용전 무조건 물리 주소에 맵핑되어야한다.**


## Dynamic Loading
> * 모든 프로그램에 물리 메모리를 할당해주기에는 물리 메모리의 크기가 한정적이다. 하지만 Dynamic loading기술을 이용하면 보다 메모리를 효율적으로 관리할 수 있다.
> * Dynamic loading은 디스크에 relocatable load format형태로 루틴을 저장하고 루틴을 호출할 때서야 메모리에 로드한다.
> * Dynamic loading은 OS차원이 아닌 사용자가 구현 하는 기능이다. 하지만 OS는 사용자의 편의를 위해서 라이브러리를 제공하고있다.

동적 로딩의 단계는 아래와 같다.
1. 메인 프로그램이 메모리에 로드되어 실행중이다.
2. 루틴은 다른 루틴을 호출하는데 이때 루틴이 로드된 상태인지 확인한다.
3. 로드되어 있지 않다면 relocatable linking loader가 호출되어 루틴을 메모리에 로드 하고 프로그램의 주소 테이블(address tables)을 수정한다.

위 방법의 장점은 루틴들이 필요할 때만 로드된다는 것 이다. 이 방법은 오류 처리 루틴같이 코드양이 방대하지만 자주 호출되지 않는 경우에 적합하다. 하나의 프로그램이 필요로 하는 총 메모리양이 크더라도 Dynamic loading은 적은 양의 메모리를 사용할 수 있게 해준다.


## Dynamic Linking and Shared Libraries
> * Dynamically linked libraries는 사용자 프로그램이 실행될 때 링크된 시스템 라이브러리들이다.
> * Dynamic loading과 비슷하지만 Dynamic linking는 로딩(loading)이 실행 시점(Execution time)까지 연기되는 차이가 있다.


일부 OS들은 **Static linking**기능을 지원한다. 이 방법은 시스템 라이브러리들을 다른 오브젝트 모듈같이 취급하여 로더(loader)가 시스템 라이브러리들을 이진 프로그램 파일에 합친다. 이 방법은 각 프로그램들이 공통적으로 사용하는 라이브러리들을 각자 카피해서 갖고있으므로 디스크와 메인 메모리의 용량 낭비가 심각해진다.

**Dynamic linking**에서 각 이진 프로그램 파일은 라이브러리 루틴 참조를 위한 **stub**라는 아주 작은 코드 조각을 가지고 있다.
stub는 메모리에 존재하는 라이브러리를 탐색하거나 존재하지 않는 라이브러리를 로드하는 방법을 알려주는 기능을 한다.    
Stub가 실행되면 꼭 필요한 루틴들이 존재하는지 확인을 한다. (없으면 로드한다.) 그 후 stub는 스스로의 주소를 필수 루틴의 주소로 교체하고 필수 루틴을 실행한다. 다음에 또 필수 루틴을 참조하는 상황이 발생하면 메모리에 로드할 필요 없이 바로 저장된 메모리 주소에 접근한다.  (각종 언어 라이브러리를 사용할 때 유용하다. C언어에서 <stdio>나 <iostream>같은 라이브러리를 매번 로드할 필요는 없지않는가?)

위 방법을 사용하는 경우 라이브러리 버전 문제가 발생할 수 있다 (Dynamic linking을 사용하지 않으면 라이브러리가 업데이트 될 때마다 새로 링크를 해줘야한다.). 프로그램에서 사용하던 라이브러리가 업데이트 되면서 특정 함수를 지원하지 않거나 내부 동작이 바뀐다면? (호환성 유지때문에 거의 그러지는 않지만 그렇다고 가정해보자) 프로그램은 먹통이 될 것이다. 때문에 라이브러리는 프로그램 호환성을 위해 버전 정보를 프로그램과 라이브러리에 포함시킨다. 새롭게 컴파일 하는 프로그램만 업데이트 된 라이브러리를 사용하고 업데이트 전에 컴파일한 프로그램들은 이전 버전을 계속 사용한다. 이 방법을 **Shared libraries**라 칭한다.

**Dynamic linking, Shared libraries는 Dynamic load와 달리 OS의 지원이 필요하다. 이유는 각 프로그램의 메모리 공간에 접근 권한을 가진것은 OS뿐이기 때문이다.**





[log]
- 2020/08/31 ~Address binding까지 작성
- 2020/09/01 Logical versus physical address space 작성
- 2020/09/02 ~ 동적링크, 공유 라이브러리 작성