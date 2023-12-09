## 3.10 Combining Control and Data in Machine-Level Programs
### 3.10.3 Out-of-Bounds Memory References and Buffer Overflow
- C에서는 배열 참조 시 범위를 체크하지 않음
- 로컬 변수는 스택에 저장되는데, 이 스택에 보존용 레지스터 값, 리턴 주소와 같은 상태 정보도 함께 저장됨
- 배열의 범위를 벗어나는 원소에 대한 쓰기 작업으로 인해 스택에 저장된 중요한 정보가 손상될 수 있음(Ex. 레지스터 값 reload, 스택이 변경된 상태에서 `ret` 인스트럭션 실행)
- *Buffer Overflow*: 문자열을 저장하기 위해 스택에 할당된 공간보다 큰 문자열이 쓰여질 때 발생하는 현상

#### Example: Buffer Overflow
``` c
/* Implementation of library function gets() */
char *gets(char *s) {
	int c;
	char *dest = s;
	while ((c = getchar()) != ’\n’ && c != EOF)
		*dest++ = c;
	if (c == EOF && dest == s)
		/* No characters read */
		return NULL;
	*dest++ = ’\0’; /* Terminate string */
	return s;
}

/* Read input line and write it back */
void echo() {
	char buf[8]; /* Way too small! */
	gets(buf);
	puts(buf);
}
```
- `gets` 함수: 전체 문자열을 저장할 수 있는 공간이 충분히 할당되었는지 알 수 없음
- `echo` 함수: 의도적으로 버퍼의 크기를 매우 작게 설정(out-of-bounds 쓰기를 발생하기  위함)

``` sh
void echo()
  echo:
  subq $24, %rsp 	# Allocate 24 bytes on stack
  movq %rsp, %rdi 	# Compute buf as %rsp
  call gets 		# Call gets
  movq %rsp, %rdi 	# Compute buf as %rsp
  call puts		    # Call puts
  addq $24, %rsp 	# Deallocate stack space
  ret 			    # Return
```
1. `buf` 배열을 저장하기 위해 스택에서 24바이트를 할당 
2. `buf`의 시작 주소를 계산하여 `%rdi` 레지스터에 저장
3. `gets` 함수를 호출하여 입력 문자열을 받아와서 `%rdi` 레지스터에 저장된 주소(`buf`의 시작 주소)에 저장
4. `buf`의 시작 주소를 다시 계산하여 `%rdi` 레지스터에 저장합니다. 이는 `puts` 함수의 인자로 사용됩니다. 
5. `puts` 함수를 호출하여 `%rdi` 레지스터에 저장된 주소에 있는 문자열을 출력
6. 할당했던 스택 공간을 반환

![[Figure 3.40.png|400]]
- `echo` 함수가 실행되는 동안의 스택 구성
- 문자 `buf`는 스택의 top에 위치함

| Characters typed | Additional corrupted state|
|---|---|
|0-7|None|
|9-23|Unused stack space|
|24-31|Return address|
|32+|Saved state in caller|
- 문자형 배열 `buf`로 범위를 초과해서 쓰면, 프로그램의 상태가 손상됨
- 23개 문자까진 심각한 결과가 발생하지 않지만, 이 이상의 경우 리턴 포인터 값과 caller 함수의 영역이 손상됨

#### Buffer Overflow Attack
- 공격자가 *exploit code*(공격자가 원하는 명령을 실행하는 코드)와 이 코드가 저장된 메모리 위치를 가리키는 정보를 포함한 데이터를 프로그램에 입력하여 발생
- Ex) exploit code가 운영체제의 명령을 실행하도록 함
1. Code Injection Attack
	- 버퍼 오버플로우를 이용하여 exploit code를 주입하고, 프로그램의 실행 흐름을 그 코드로 리디렉션하여 프로그램이나 시스템을 제어
2. Return Oriented Programming Attack(ROP)
	- 직접 코드를 주입하는 대신, 프로그램의 기존 코드 조각 *gadget*을 재사용
	- 버퍼 오버플로우를 통해 gadget들의 주소를 스택에 배치하고, 프로그램의 실행 흐름을 이 가젯들로 리디렉션하여 원하는 동작을 수행
	- `ret` 인스트럭션으로 발생
	- 각 gadget의 마지막 `ret`은 다음 gadget을 시작
	- Challenge for hackers
		- 스택 랜덤화는 버퍼 위치를 예측하기 어렵게 만듦
		- 스택을 실행 불가능하게 표시함으로써, 이진 코드를 삽입하는 것을 어렵게 함
	- Alternatives
		- 기존의 코드를 사용(Ex. `stdlib`의 라이브러리 코드)
		- 전체적으로 원하는 결과를 달성하기 위해 코드 조각들을 연결
	- Construct program from gadgets
		- `ret`(0xc3) 인스트럭션으로 끝나는 명령어의 시퀀스
		- gadget의 위치가 프로그램의 실행 사이에 변경되지 않음
		- 기계어로 작성된 코드들을 포함해, 프로세서에 의해 직접 실행될 수 있음

### 3.10.4 Thwarting Buffer Overflow Attacks
#### Avoid Overflow Vulnerabilities in Code
- `fgets` 대신 `gets` 사용
	- `gets`: `NULL` 문자를 만날 때까지 입력을 받음 
	- `fgets`: 읽을 최대 문자 수를 지정하여 buffer overflow 방지
- `strncpy` 대신 `strcpy` 사용
	- `strcpy`: `NULL` 문자를 만날 때까지 복사를 수행
	- `strncpy`: 복사할 최대 문자 수를 지정하여 buffer overflow 방지
- `%s` 변환 지정자로 `scanf` 사용하지 않기
	- `%s` 변환 지정자를 사용하면 `scanf`는 NULL 문자를 만날 때까지 입력 받음
	- `fgets` 함수를 사용하거나, `%ns`를 사용하여 읽을 최대 문자 수를 지정

#### Stack Randomization
- 프로그램이 실행될 때마다 스택의 위치를 변경하여, 공격자가 스택 주소를 예측하기 어렵게 만드는 기법
- 프로그램이 시작될 때 스택에 `alloca` 함수를 사용하여 0에서 *n*바이트 사이의 랜덤한 크기의 공간을 할당. 이 공간은 프로그램에서 사용되지 않지만, 이로 인해 프로그램의 매 실행마다 모든 후속 스택 위치가 변하게 됨.
- 스택 랜덤화는 주소 공간 레이아웃 랜덤화(Address-Space Layout Randomization, ASLR)라는 더 넓은 범주의 기법 중 하나
	- *ASLR*: 프로그램이 실행될 때마다 프로그램의 다양한 부분(프로그램 코드, 라이브러리 코드, 스택, 전역 변수, 힙 데이터 등)이 메모리의 다른 영역에 로드되도록 하는 기법
- 한계: 공격자가 'nop sled'를 설정하고 끊임없이 무작위로 주소를 변경함으로써 이러한 랜덤화를 극복할 수 있음. 완전한 보호를 제공하지 못함.
	- *nop sled*: 여러 개의 `nop`(No Operation) 인스트럭션을 실제 exploit code 앞에 포함시키는 코드 블록

#### Stack Corruption Detection
- 스택이 손상되었을 때 이를 감지하는 기법
	- 배열의 범위를 넘는 쓰기가 발생했을 때, 그것이 해로운 영향을 미치기 전에 감지하려고 시도
- stack protector code: 지역 버퍼와 스택의 나머지 스택 상태 값 사이에 특별한 'canary 값'을 스택 프레임에 저장
	- *canary value*(guarded value): 프로그램이 실행될 때마다 랜덤하게 생성되어 공격자가 값을 쉽게 추정할 수 없음
	- canary 값을 특별한 세그먼트에 저장하여 'read only'로 표시할 수 있으며, 공격자가 값을 변경할 수 없게됨
- 프로그램은 레지스터 상태를 복원하고 함수에서 리턴하기 전에, canary 값이 이 함수나 그것이 호출한 함수의 어떤 작업에 의해 변경되었는지 확인하여 프로그램을 오류와 함께 중단시킴
- 최신 gcc 버전에서는 함수가 stack overflow에 취약한지 판단하려 하고, 감지 기능을 자동으로 삽입함

``` sh
# void echo()
echo:
  subq $24, %rsp 		# Allocate 24 bytes on stack
  movq %fs:40, 			# %rax Retrieve canary
  movq %rax, 8(%rsp) 	# Store on stack
  xorl %eax, %eax 		# Zero out register
  movq %rsp, %rdi 		# Compute buf as %rsp
  call gets 			# Call gets
  movq %rsp, %rdi 		# Compute buf as %rsp
  call puts 			# Call puts
  movq 8(%rsp), %rax 	# Retrieve canary
  xorq %fs:40, %rax 	# Compare to stored value
  je .L9 			    # If =, goto ok
  call __stack_chk_fail # Stack corrupted!
.L9: 				# ok:
  addq $24, %rsp 		# Deallocate stack space
  ret
```
- `movq %fs:40, %rax`: fs 세그먼트 레지스터의 40번째 오프셋에 저장된 카나리 값을 %rax 레지스터에 저장
- `movq %rax, 8(%rsp)`: %rax 레지스터에 있는 카나리 값을 스택 프레임의 8번째 오프셋 위치에 저장
- `movq 8(%rsp), %rax`: 이전에 저장한 카나리 값을 %rax 레지스터로 복사
- `xorq %fs:40, %rax`: 저장된 카나리 값을 원래의 카나리 값과 비교(두 값이 같다면 결과는 0)
- `je .L9`: 만약 xor 연산의 결과가 0이라면(즉, 카나리 값이 변하지 않았다면), .L9 레이블로 이동
- `call __stack_chk_fail`: 카나리 값이 변경되었다면(즉, 스택 오버플로우 공격이 발생했다면), 스택 손상을 알리는 `__stack_chk_fail` 함수를 호출

#### Limiting Executable Code Regions
- 실행코드가 저장되는 메모리 영역을 제한하는 기법
- 공격자가 실행코드를 시스템에 추가하지 못하게 함

---
