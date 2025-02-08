---
title: "[wip](Review) Google File System (2003) "
date: 2025-01-29T19:34:29+09:00
showToc: false
---

이 포스트는 [The Google File System, 2003](https://research.google/pubs/the-google-file-system) 논문을 읽고 정리한 내용입니다.

### 개요

Google File System (GFS)는 구글에서 개발한 확장 가능한 분산 파일 시스템으로, 구글의 데이터 처리 요구사항을 충족하기 위해 설계되었습니다.

이 논문을 읽으며 GFS의 설계 방식을 이해하고, 구글이 어떻게 Fault tolerance and Scalable distributed system을 구축했는지 알아보려고 합니다.

---
GFS는 구글 내부의 파일 처리를 위해 개발되었기 때문에 다음과 같은 조금 특별한 가정 아래에 설계되었습니다. 
- GFS는 저비용 상용 하드웨어(commodity hardware)로 구성됩니다. 이러한 하드웨어는 고장이 빈번하므로, 하드웨어 결함이 일상적인 것으로 간주됩니다. 이를 위해 시스템은 스스로 오류를 감지하고 자동으로 복구할 수 있어야 합니다. 

- 시스템은 주로 100MB 이상의 대규모 파일을 저장합니다. 수 GB에서 수 TB에 이르는 파일도 흔하며, 이러한 대규모 파일이 효율적으로 처리되어야 합니다. 

- 대부분의 파일은 append 방식으로 데이터를 추가한 뒤 읽기 전용으로 사용됩니다. 읽기 패턴은 주로 순차적 스트리밍 읽기(sequential streaming read)로 이루어집니다. 

- GFS는 여러 클라이언트가 동시에 데이터를 추가(append)할 수 있는 기능을 제공해야 하며, 이때 데이터 무결성과 원자성을 보장해야 합니다. 

- 높은 처리량(high bandwidth)은 짧은 지연 시간(low latency)보다 더 중요합니다. 이는 GFS가 대규모 데이터 처리 애플리케이션을 위해 설계되었음을 보여줍니다. 
 \
  <br/> \
  <br/> 
### GFS의 기본 구조
GFS에서는 파일이 고정된 크기의 청크(chunk)로 나뉘어 저장됩니다. 각 청크는 64MB 크기이며, 고유한 ID인 chunk handle을 통해 식별됩니다.  
Chunkserver(청크 서버)는 이러한 청크를 로컬 디스크에 저장하고, 클라이언트의 요청에 따라 읽기와 쓰기 연산을 수행합니다. 데이터의 안정성을 위해 GFS는 각 청크 마다 3개의 복제를 가지도록 설정합니다.

{{< figure src="gfs-1.png" caption="drawn by excalidraw">}}

#### 1. 마스터 (Master)
마스터는 시스템의 메타데이터를 관리하는 중앙 서버입니다. 주요 역할은 다음과 같습니다:
- 파일 및 청크 메타데이터 관리: 파일 네임스페이스(namespace), 파일-청크 매핑(chunk mapping) 등을 저장
- 청크 할당 및 복제 관리: 새로운 청크를 할당하고, 데이터 안정성을 유지하기 위해 복제본을 관리
- 청크 서버 모니터링: 주기적으로 청크 서버와 통신하여 상태를 확인하고, 가비지 컬렉션(garbage collection) 수행

마스터는 단일 서버로 동작하기 때문에 지나친 요청은 병목 현상을 일으킬 수 있습니다. GFS에서 마스터는 데이터 흐름에 직접 관여하지 않음으로써 병목 현상을 방지합니다.
즉, 클라이언트가 데이터를 읽거나 쓸 때 직접 청크 서버와 통신하여 성능을 유지할 수 있도록 설계되었습니다.
  <br/> \
  <br/> 

#### 2. 청크 서버 (Chunkserver)
청크 서버는 클라이언트가 요청하는 데이터를 로컬 디스크에 저장하고 제공하는 역할을 합니다.
- 각 청크는 64MB 의 고정된 크기를 가집니다.
- 클라이언트는 가장 가까운 청크 서버에 직접 요청하여 데이터를 효율적으로 읽고 쓸 수 있습니다.
  <br/> \
  <br/> 

#### 3. 클라이언트 (Client)
클라이언트는 마스터로부터 메타데이터(청크 위치 정보)를 가져온 후, 직접 청크 서버와 데이터를 주고받습니다.
이를 통해 마스터의 부하를 줄이고, 요청을 여러 청크 서버로 분산시켜 성능을 최적화합니다.

  <br/> \
  <br/> 

#### GFS에서의 데이터 읽기/쓰기 흐름
클라이언트가 데이터를 읽거나 쓸 때의 기본적인 동작 방식은 다음과 같습니다:

1. 클라이언트 → 마스터:  
   클라이언트는 파일명과 chunk index를 이용해 마스터에 해당 청크의 위치를 요청합니다.
2. 마스터 → 클라이언트:  
   마스터는 해당 청크가 저장된 청크 서버 목록과 chunk handle(고유 ID)을 클라이언트에게 전달합니다.
3. 클라이언트 → 청크 서버:  
   클라이언트는 가장 가까운 청크 서버에 직접 읽기/쓰기 요청을 보냅니다.

이후의 데이터 동기화 과정은 밑에서 다루겠습니다.

---
### Consistency
GFS 는 완화된 일관성(Relaxed Consistency) 모델을 사용합니다. \
완화된 일관성 모델은 전통적인 강한 일관성(Strong Consistency) 모델과 대비되는 개념으로 \
시스템이 일정한 수준의 일관성을 보장하지만 완전한 동기화를 강제하지는 않는 모델입니다.

즉, 데이터 업데이트가 모든 클라이언트에게 즉시 동일하게 반영되지 않을 수 있으며 \
각 클라이언트가 일관되지 않은 데이터를 읽을 수도 있는 상황을 허용하는 방식입니다.

GFS에서의 일관성 모델에 대해 알아보기 전에 논문에서 나오는 용어인 `Mutation` 에 대해서 이야기하고 넘어가겠습니다.

> GFS에서 mutation은 쓰기(Write)나 추가(Append) 작업과 같이 청크의 내용 또는 메타데이터를 변경하는 작업을 의미합니다.  
> 
> mutation 연산은 모든 레플리카에 적용되며, 성공하거나 실패할 수 있습니다.
> 
> 또한, mutation은 동시에 여러 개의 클라이언트에 의해 수행될 수도 있고, 직렬(순차)로 수행될 수도 있습니다.
> 
> 이러한 세 가지 차원(성공/실패, 직렬/동시)에 따라 파일 region은 다양한 상태를 가집니다.  

아래 표는 mutation 이후 파일 region 상태를 나타냅니다.

{{< figure src="gfs-2.png" caption="The Google File System(2003)">}}

- **consistent**: 모든 클라이언트가 모든 레플리카에서 동일한 데이터를 볼 수 있는 상태
- **defined**: consistent 상태이면서, mutation의 결과가 레플리카에 반영된 상태

mutation이 순서대로 들어오고 성공하는 경우, region은 defined 상태가 됩니다. 동시에 mutaion이 작업되는 경우 region은 consistent 상태가 됩니다.

#### Record Append의 일관성
GFS의 Record Append 연산은 여러 클라이언트가 동시에 데이터를 추가할 수 있도록 설계되었습니다.  
이 연산은 모든 레플리카에서 최소 1회 이상 수행됨을 보장합니다.
일부 레플리카에서 연산이 실패하는 경우, GFS는 자동으로 다시 연산을 수행하여 데이터의 변경을 보장합니다.

그러나 이 과정에서 일부 레플리카에 데이터가 중복 저장될 수도 있습니다. \
따라서 애플리케이션이 이를 감안하고 중복을 감지하고 제거하는 로직을 구현해야 합니다.

#### In Application

위의 동시성 모델을 따른다면, 중복된 데이터가 레플리카에 저장되어 있기도 하고, 오래된 레플리카가 존재하기도 할 것 같습니다. \
그렇기 때문에 어플리케이션에서 이러한 것들을 다루기 위해 다음과 같은 기술들이 필요합니다.
1. Overwrite(덮어쓰기) 대신 Append(추가) 연산 사용  
   - 덮어쓰기를 하면 데이터가 Undefined 상태가 될 수 있으므로 Append 방식을 사용하여 항상 새로운 데이터가 추가되도록 설계합니다.

2. Self-Validating & Self-Identifying Record 기록  
   - 각 레코드에 고유한 ID와 체크섬(Checksum)을 포함하여 데이터를 읽을 때 유효성을 검증할 수 있도록 합니다.  
   - 예를 들어, 로그 파일에 각 로그 항목마다 타임스탬프(Timestamp)와 고유 식별자(UUID)를 포함하면 중복 데이터를 쉽게 식별할 수 있습니다.

3. Checkpointing (체크포인트 저장) 기법 활용  
   - 데이터 처리 도중 특정 시점의 상태를 저장하여 만약 중복 데이터가 발생하더라도 정상적인 상태로 되돌릴 수 있도록 설계합니다.  
   - 예를 들어, 분산 데이터 처리 시스템에서는 일정 주기마다 진행 상태를 저장하여 중복된 데이터가 있더라도 동일한 처리를 반복하지 않도록 할 수 있습니다.

### System Interactions
이번 장에서는 클라이언트 / 마스터 / 청크 서버 구조에서 Data mutation, Atomic record append, Snapshot을 구현한 방식을 설명합니다.

#### Data mutation

GFS는 마스터의 오버헤드를 최소화하도록 설계되어 있습니다. 이를 위해 레플리카 중 하나에 Lease를 부여하여 consistent mutation order를 유지합니다. \
lease가 부여된 레플리카를 primary라고 합니다. primary는 청크에 대한 mutation의 순서를 지정하고 다른 모든 레플리카는 이 순서에 따라 mutation 연산을 수행합니다. \

예를 들어 하나의 청크에 동시에 많은 mutation 요청이 들어왔을 때, 각 레플리카에 mutation이 들어온 순서가 다양하게 있을 수 있습니다. 이를 일관된 순서로 만들기 위해 primary가 mutation의 순서를 지정하고, 그것을 모든 레플리카가 따름으로써 하나의 청크에 대한 모든 레플리카의 데이터가 동일함이 보장됩니다.

이러한 역할을 마스터가 아닌, 마스터가 지정한 하나의 레플리카가 수행함으로써 마스터의 관리 오버헤드가 최소화됩니다.

mutation이 반영되는 과정은 다음과 같습니다.

{{< figure src="gfs-3.png" caption="The Google File System(2003)">}}

1. 클라이언트는 마스터에 어떤 청크 서버가 Lease로 지정되어 있는지 요청합니다.
2. 마스터는 요청한 청크에 대한 primary와 나른 레플리카를 리턴합니다.
3. 클라이언트는 데이터를 모든 레플리카에 전달합니다. (이때 어떤 레플리카에 먼저 전달할지와 같은 순서는 중요하지 않습니다.)
4. 모든 레플리카에 데이터를 전송한 이후에 클라이언트는 primary에 mutation 요청을 보냅니다.
    - primary는 여러 클라이언트로부터 동시에 요청을 받을 수 있습니다.
    - 들어온 요청들을 하나의 순서로 만듭니다.(직렬화)
5. primary는 mutation 요청을 다른 모든 레플리카에 전달합니다. 각 레플리카는 동일한 순서로 mutation 연산을 수행합니다.
6. 레플리카에서 연산이 끝나면 primary에 끝났음을 알립니다.
7. primary는 클리아언트에게 결과를 알립니다. 연산 중 오류가 발생한 경우 region은 unconsistent 상태로 남겨지고, 클아이언트는 3~7 과정을 반복합니다.

그림의 3번 과정을 보면, 클라이언트는 모든 레플리카에 데이터를 한 번에 전달하지 않고 순차적으로 전달하는 것을 볼 수 있습니다. GFS는 네트워크 병목을 막기 위해 가장 가까이 있는 노드에 데이터를 전달합니다. 예를 들어 클라이언트가 S1 ~ S4 레플리카에 데이터를 전송해야 할 때, 클라이언트는 가장 가까이 있는 S1 레플리카에 데이터를 전송하고, S1은 S2에, S2는 S3, S3는 S4에 데이터를 전달합니다. 이를 통해 노드의 처리량을 높일 수 있게 됩니다.

#### Atomic record append
GFS는 record append라고 하는 atomic append 연산을 제공합니다. 일반적으로 쓰기 연산은 클라이언트가 쓰고자 하는 위치(offset)과 데이터를 같이 전달합니다. record append 요청에서 클아이언트는 오프셋 없이 데이터만 전달합니다. 운영체제의 O_APPEND 연산과 같이 작동하며, 경쟁 상태 없이 동시에 쓰기 요청을 처리할 수 있습니다. 클라이언트에서 동기화에 대한 신경을 덜 쓰고 요청에만 집중할 수 있게 됩니다. 

record append 연산은 위에서 설명한 Data mutation 과정에서 일부 로직이 추가됩니다.

```
클라이언트는 데이터를 모두 레플리카에 전달한 다음(3번, 4번 과정) primary에게 알립니다.

primary는 현재 청크에 데이터를 모두 추가하는 게 가능한지 확인합니다. 왜냐하면 data mutation은 청크 단위로 쓰기 요청을 처리하는 연산이기 때문입니다.

만약 record append 에서 요청한 데이터가 청크위 최대 크기(64mb)를 넘게 되면 일부만 처리하고, 클라이언트에 다시 요청하도록 응답을 전송합니다.

여러번의 요청으로 청크 사이즈에 딱 맞게 데이터 쓰기가 끝났을 때 성공 응답을 클라이언트에 전송합니다.
```

이 과정에서 어떤 레플리카에서 record append 연산이 실패하는 경우, 모든 레플리카가 동일한 청크를 갖지 못하게 됩니다. 대신 시스템은 클라이언트에게 다시 쓰기 요청을 보냄으로써 데이터가 최소 1번 쓰였음을 보장합니다. [Record Append의 일관성](#record-append의-일관성) 절에서 언급한 것 처럼 중복된 데이터는 애플리케이션에서 다뤄야할 부분이 됩니다.

#### Snapshot
snapshot은 다른 연산들에 영향을 최소화하면서 파일 혹은 소스의 복사본을 만드는 연산입니다. 
GFS는 copy-on-write 방식을 통해 snapshot을 구현합니다.

>Copy-on-write (COW), also called implicit sharing or shadowing is a resource-management technique used in programming to manage shared data efficiently. Instead of copying data right away when multiple programs use it, the same data is shared between programs until one tries to modify it. If no changes are made, no private copy is created, saving resources. A copy is only made when needed, ensuring each program has its own version when modifications occur. This technique is commonly applied to memory, files, and data structures.
>
>[Copy-on-write Wikipedia](https://en.wikipedia.org/wiki/Copy-on-write)

snapshot의 절차는 다음과 같습니다.
1. 마스터가 snapshot 요청을 받으면, 해당 청크에 대한 Lease를 갱신합니다.
2. 마스터는 이후 들어오는 연산들을 디스크에 기록합니다.
3. 이렇게 생성된 snapshot은 동일한 청크를 가리킵니다.

snapshot 연산 이후 클라이언트가 쓰기 요청을 보내면 다음과 같은 절차를 통해 청크가 업데이트됩니다.
1. 클라이언트가 마스터에 C1 청크에 대한 Lease 정보를 획득하기 위해 조회 요청을 보냅니다.
2. 마스터는 C1 청크의 reference count 가 1보다 큰 것을 확인합니다.
3. 마스터는 C2 라는 새로운 청크 핸들을 만들고, C1을 저장하는 레플리카에 C1을 복사한 C2라는 청크를 만들도록 요청합니다. ( 이 복사는 각 청크 서버의 로컬에서 이루어집니다. )
4. 이후 클라이언트의 쓰기 요청은 C2에서 이루어집니다.

이러한 copy-on-write 방식을 도입함으로써 GFS는 더 효율적으로 리소스를 관리할 수 있게 됩니다. 
청크를 바로 복사하는 대신 수정이 발생할 때까지 동일한 청크를 공유합니다. 
복사본은 필요할 때만 만들어지므로 수정이 발생할 때 각 프로그램마다 고유한 버전을 유지할 수 있습니다.

### 마무리

#### 참조

- https://research.google/pubs/the-google-file-system
- https://en.wikipedia.org/wiki/Copy-on-write
- https://hemantkgupta.medium.com/insights-from-paper-the-google-file-system-1c9d7b32c8cc