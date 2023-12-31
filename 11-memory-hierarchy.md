# 6. The Memory Hierarchy
---
## 6.2 Locality


## 6.3 The Memory Hierarchy
- Storage technology
	- 빠른 저장 기술들은 느린 기술들보다 바이트당 비용이 더 많이 들고 용량이 적음
	- CPU와 메인 메모리의 속도 차이는 점점 벌어지고 있음
- Computer software
	- 잘 작성된 프로그램들은 좋은 *locality*을 보임

![[Figure 6.21.png|500]]
- 컴퓨터 시스템의 메모리 계층: 빠르고 비싸며 용량이 작은 CPU 레지스터에서 시작해, 접근 시간은 느려지고, 비용은 저렴해지며, 용량은 커지는 형태로 구성
### 6.3.1 Caching in the Memory Hierarchy
- *cache*: 크고 느린 디바이스에 저장된 데이터 객체를 준비하기 위한 준비영역으로 사용되는 작고 빠른 저장장치
	- CPU와 메인 메모리(DRAM) 간의 속도 차를 극복하고, 데이터 접근 시간을 줄이기 위해 사용
- *caching:* 캐시를 활용하여 데이터의 접근 속도를 향상시키는 과정

메모리 계층의 핵심 개념은 각 계층 k에서 더 빠르고 작은 저장 장치가 더 크고 느린 k+1 계층의 저장 장치를 캐시로 사용한다는 것입니다. 즉, 계층구조에서 각 계층은 바로 아래 계층의 데이터 객체를 캐시에 저장합니다. 예를 들어, 로컬 디스크는 네트워크를 통해 원격 디스크에서 검색된 파일(웹 페이지 등)을 캐시로 사용하고, 메인 메모리는 로컬 디스크의 데이터를 캐시로 사용하며, 이러한 과정이 가장 작은 캐시인 CPU 레지스터에 이르기까지 계속됩니다.

메모리 계층에서 캐싱의 일반적인 개념을 보여주는 그림 6.22에서, k+1 계층의 저장 공간은 '블록'이라는 연속적인 데이터 객체로 나누어져 있습니다. 각 블록은 고유한 주소나 이름을 가지며 다른 블록들과 구분됩니다. 블록들은 고정 크기일 수도 있고(일반적인 경우), 변수 크기일 수도 있습니다(예: 웹 서버에 저장된 원격 HTML 파일).

또한, k 계층의 저장 공간은 k+1 계층의 블록과 동일한 크기의 블록으로 이루어진 더 작은 집합으로 나누어져 있습니다. 어느 시점에서든, k 계층의 캐시는 k+1 계층의 블록 중 일부의 복사본을 포함하고 있습니다.

데이터는 항상 블록 크기의 전송 단위로 k 계층과 k+1 계층 사이를 오고갑니다. 계층 구조에서 인접한 두 계층 사이의 블록 크기는 고정되어 있지만, 다른 계층 쌍은 다른 블록 크기를 가질 수 있습니다. 일반적으로, 계층에서 CPU와 더 멀리 있는 장치는 접근 시간이 더 길며, 이로 인한 더 긴 접근 시간을 분산하기 위해 더 큰 블록 크기를 사용하는 경향이 있습니다.


