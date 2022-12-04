# 운영체제

# 프로세스
## 프로그램과 프로세스 차이
프로그램(prgram)
- 하드디스크 등의 저장 매체에 저장
- 실행 파일의 형태

프로세스(prcess)
- 프로그램이 메모리에 적재되어 실행 중인 상태
  - 필요한 모든 자원을 할당 받는다.
  - 자원이란 코드/데이터/스택/힙 공간을 말한다.

## 프로세스 특징
- 운영체제는 프로그램을 메모리에 적재하고 프로세스로 다룬다.
- 운영체제는 프로세스에게 실행에 필요한 메모리를 할당한다. 이곳에 코드와 데이터 등을 적재한다.
- 프로세스들은 서로 독립적인 메모리 공간을 가진다. 다른 프로세스의 영역에 접근 불가능.

## 프로세스 관리
프로세스 생성에서 종료까지 관리는 모두 커널에 의해 이루어진다. 커널 영역에 프로세스 테이블을 만들고, 프로세스의 목록을 관리한다.

관리 내용
- 프로세스 생성/실행/일시중단/재개/중단
- 프로세스 정보 관리
  - 프로세스의 메모리 위치
  - 크기 
- 프로세스마다 고유한 번호(프로세스 ID)를 할당
- 프로세스 통신
- 프로세스 동기화
- 프로세스 컨텍스트 스위칭

## 프로그램의 다중 인스턴스
한 프로그램을 여러 번 실행시켜 다중 인스턴스를 생성할 수 있다. 운영체제는 프로그램을 실행할 때마다 독립된 프로세스를 생성한다. 각 프로세스는 독립된 메모리 공간이 할당된다. 

## 프로세스를 구성한 4개의 메모리 영역
### 코드(code) 영역
실행될 프로그램 코드가 적재되는 영역
- 사용자가 작성한 모든 함수 코드
- 사용자가 호출한 라이브러리 함수 코드

### 데이터(data) 영역
프로그램에서 고정적으로 만든 변수 공간으로 프로세스 적재 시 할당, 종료 시 소멸된다.
- 전역 변수 공간
- 정적 데이터 공간
- (사용자 프로그램과 라이브러리 포함)

### 힙(heap) 영역
프로세스가 실행 도중 동적으로 사용할 수 있도록 할당된 공간
- malloc() 등으로 할당받은 공간
  
힙 영역에서 아래 번지로 내려가며 할당된다.

### 스택(stack) 영역
함수가 실행될 때 사용될 데이터를 위해 할당된 공간
- 매개변수
- 지역변수
- 리턴값

함수는 호출될 때, 스택 영역에서 위쪽으로 공간을 할당한다. 함수가 return 하면 할당된 공간이 반환된다. 함수 호출 외에 프로세스에서 필요 시 사용가능하다.

## 프로세스 주소 공간
프로세스가 실행 중에 접근할 수 있도록 허용된 주소의 최대 범위이다. 

프로세스 주소 공간은 논리 공간(가상 공간)이다. 0번지에서 시작하여 연속적인 주소를 갖는다.

프로세스 주소 공간은 CPU가 액세스할 수 있는 전체 크기이다. (32 비트 CPU의 경우, 4GB)

### 프로세스 공간과 프로세스 현재 크기
프로세스 주소 공간의 크기는 프로세스의 현재 크기와 다르다.
- 프로세스 주소 공간 크기: 프로세스가 액세스할 수 있는 최대 크기
- 프로세스 현재 크기: 적재된 코드 + 전역 변수 + 힙 영역에서 할당받아 사용중인 동적 메모리 공간 + 현재 스택 영역에 저장된 데이터 크기

### 사용자 공간과 커널 공간
프로세스 주소 공간은 2부분으로 나뉘어진다.

사용자 공간
- 프로세스의 코드, 데이터, 힙, 스택 영역이 할당되는 공간
- 코드와 데이터 영역의 크기는 프로세스 시작 시 결정된다.
- 힙과 스택의 영역 크기는 정해져 있지 않다.
- 힙 영역은 아래로 자라고, 스택은 위로 자란다.
  
커널 공간
- 프로세스가 시스템 호출을 통해 이용하는 커널 공간
- 커널 코드, 커널 데이터, 커널 스택(커널이 실행될 때)
- 커널 공간은 모든 사용자 프로세스에 의해 공유된다.
  
### 가상 공간
프로세스 주소 공간은 사용자나 개발자가 보는 관점이다.
- 자신이 작성한 프로그램이 0번지부터 시작하여
- 연속적인 메모리 공간에 형성되고,
- CPU가 액세스할 수 있는 최대 크기의 메모리가 설치되어 있다고 상상

실제 상황
- 실제 물리 메모리 크기는 프로세스 주소 공간보다 작을 수 있다.
- 프로세스의 코드, 데이터, 힙, 스택 영역은 물리 메모리에 흩어져 저장된다. 연속적인 메모리 공간이 아니다.

# 커널의 프로세스 관리
## 프로세스 테이블과 프로세스 제어 블록
프로세스 테이블(Process Table) : 시스템의 모든 프로세스들을 관리하기 위한 표
- 시스템에 1개만 존재한다.
- 구현 방식은 OS마다 다르다.

프로세스 제어 블록(Process Control Block, PCB) : 프로세스에 관한 정보를 저장하는 구조체
- 프로세스당 1개씩 존재
- 프로세스가 생성될 때 만들어지고 종료되면 삭제된다.
- 커널에 의해 생성되고 저장, 읽혀지는 등 관리된다.

프로세스 테이블과 프로세스 제어 블록의 위치
- 커널 영역, 커널 코드(커널 모드)만이 액세스 가능

# 프로세스 생명 주기와 상태 변이(state change)
## 프로세스의 생명 주기
프로세스는 탄생에서 종료까지 여러 상태로 바뀌면서 실행된다.

<img src="../imgs/os-state-change.png">

## 프로세스의 상태
### New [생성 상태]
- 프로세스가 생성된 상태
- 메모리 할당 및 필요한 자원을 적재한다.

### Ready [준비 상태]
- 프로세스가 스케줄링을 기다리는 준비 상태
- 프로세스는 준비 큐에서 대기한다.
- [dispatch] 스케줄링되면 Running 상태로 바뀌고 CPU에 의해 실행된다.

### Running [실행 상태]
- 프로세스가 CPU에 의해 현재 실행되고 있는 상태
- [timeout] CPU의 시간할당량(타임슬라이스,time slice)가 지나면 다시 Ready 상태로 바뀌고 준비 큐에 삽입된다.
- [block] 프로세스가 입출력을 시행하면 커널은 프로세스를 Blocked 상태로 만들고 대기 큐에 삽입한다.

### Blocked/Wait [블록 상태]
- 프로세스가 자원을 요청하거나, 입출력 요청하고(시스템 호출) 완료를 기다리는 상태
- [wakeup] 입출력이 완료되면 프로세스는 Ready 상태로 바뀌고 준비 큐에 삽입된다.

### Terminated/Zombie 상태
- 프로세스가 종료된 상태
- 프로세스가 차지하고 있던 메모리와 할당받았던 자원을 모두 반환(열어 놓은 파일 닫힘)
- Zombie 상태: 프로세스의 PCB에 남긴 종료코드를 부모 프로세스가 읽어가지 않아 완전히 종료되지 않은 상태
  - 프로세스의 테이블 항목과 PCB가 시스템에 여전히 남아있는 상태

### Terminated/Out 상태
- 프로세스가 종료하면서 남긴 종료코드를 부모 프로세스가 읽어가서 완전히 종료된 상태
- 프로세스 테이블의 항목과 PCB가 시스템에서 완전히 제거된 상태

## 프로세스 스케줄링과 컨텍스트 스위칭 
과거 OS의 실행단위는 프로세스였다. Ready 상태의 프로세스 중 실행 시킬 프로세스를 선택했다.

오늘날 OS의 실행단위는 스레드다. Ready 상태의 스레드 중 실행 시킬 스레드를 선택한다.

프로세스는 스레드들에게 공유 자원을 제공하는 컨테이너(Container)로 역할이 바뀌었다. 