## 3.6 Control
### 3.6.1 Condition Codes
CPU는 가장 최근의 arithmetic, logical operation을 설명하는 single-bit 조건 코드(condition code) 레지스터들을 운영

- *CF(Carry Flag)*: 가장 최근 연산에서 MSB를 받아 올림(carry)이 발생한 것을 표시 
	- unsigned 연산에서 overflow를 검출할 때 사용
- *ZF(Zero Flag)*: 가장 최근의 연산 결과가 0인 것을 표시
- *SF(Sign Flag)*: 가장 최근의 연산이 음수를 생성한 것을 표시
- *OF(Overflow Flag)*: 가장 최근의 연산이 2의 보수 overflow를 발생시킨 것을 표시
#### Example
``` c
// t = a + b;를 수행하기 위해 ADD 인스트럭션 사용
(unsigned) t < (unsigned) // CF: a Unsigned overflow
(t == 0) // ZF: Zero
(t < 0) // SF: Negative
(a < 0 == b < 0) && (t < 0 != a < 0) // OF: Signed overflow
```

![[Figure 3.10.png|400]]
- leaq: 주소 계산을 위한 인스트럭션으로, 조건 코드를 변경하지 않음
- logical operation: CF와 OF가 0으로 설정
- shift operation: CF가 shift되어 없어지는 마지막 비트로 설정, OF가 0으로 설정
- INC, DEC: OF와 ZF는 설정되지만, CF는 영향을 주지 않음

![[Figure 3.13.png|400]]
- 모두 레지스터에 영향을 주지 않고 조건 코드만 생성
	- *cmp_*: 두 operand(a, b)의 차(SUB)에 따라 조건 코드설정(cmp_ b,a)
		- CF: set if carry out from MSB(Used for unsigned comparisons)
		- ZF: set if a == b
		- SF: set if (a-b) < 0
		- OF: set if two’s complement overflow
	- *test_*: 두 operand를 AND 연산한 값에 따라 조건 코드 설정
		- operand들이 0인지(ZF) , 음수(SF)인지 판단하기 위해 사용
			- Ex) `testq %rax,%rax`
		- 비트 마스크를 통해 특정 비트가 설정되었는지 확인하는 데 사용
			- Ex) `testb $0x02,%al`: %al의 두 번째 비트(0x02 -> 0000 0010)가 1이 아니면 ZF를 1로 설정
		- ZF: set when a   &   b   ==   0
		- SF: set when a   &   b   <   0

### 3.6.2 Accessing the Condition Codes
![[Figure 3.14.png|500]]
- *set_*: 조건 코드의 조합에 따라 0 또는 1을 한 개의 byte에 기록
	- 접미어가 operand의 크기가 아닌 조건 코드의 어떤 조합을 사용할 것인지 나타냄
	- 일반적으로 SET 인스트럭션 실행 후 movzbl을 수행함.
	- signed: SF ^ OF == 1 -> a < b
	- unsigned: CF == 1 -> a < b
#### Example
``` sh
# int comp(data_t a, data_t b)
# a in %rdi, b in %rsi
comp:
  cmpq %rsi, %rdi # Compare a:b
  setl %al # Set low-order byte of %eax to 0 or 1
  movzbl %al, %eax # Clear rest of %eax (and rest of %rax)
  ret
```

### 3.6.3 Jump Instructions
![[Figure 3.15.png|400]]
- *j_*: 조건에 따라 dest label로 jump하여 새로운 위치로 프로그램 실행을 전환
	- *jmp*: 무조건적으로 label로 jump. 또는 operand에 저장된 값이나 저장된 값을 주소로 사용하여 메모리에 저장된 값으로 점프(indirect jump)
	- conditional jump: 조건 코드에 따라 점프를 실행. SET 인스트럭션의 조건들과 동일.
### 3.6.4 Jump Instruction Encodings
- PC relative address(상대 주소): 대상 인스트럭션과 점프 인스트럭션 바로 다음에 오는 인스트럭션 주소 차이를 인코딩. 일반적으로 이 방법을 사용.
- absolute address(절대 주소): 대상을 직접 명시하기 위해 4바이트를 사용

#### Example
``` sh
4004d0: 48 89 f8 		mov %rdi,%rax
4004d3: eb 03 jmp 		4004d8 <loop+0x8>
4004d5: 48 d1 f8 		sar %rax
4004d8: 48 85 c0 		test %rax,%rax
4004db: 7f f8 jg 		4004d5 <loop+0x5>
4004dd: f3 c3 			repz retq
```
위와 같이 상대 주소 사용시 링커 단계 후 인스트럭션들이 다른 주소에 재배치 되어도 점프 목적지 값은 바뀌지 않음

### 3.6.5 Implementing Conditional Branches with Conditional Control
if문을 conditional branch를 이용한 machine code를 C로 나타낸 코드
``` c
// if statement
if (test-expr)
	then-statement
else
	else-statement

// translate using conditional branches
t = test-expr;
if (!t)
	goto false;
then-statement
	goto done;
false:
	else-statement
done:
``` 
- 컴파일러는 `else-statement`와 `then-statement`에 대해 별도의 코드 블록을 생성함
- 이 방법은 pipeline을 통한 instruction flow에 방해되므로, 좋지 않음(old style)

### 3.6.6 Implementing Conditional Branches with Conditional Moves
![[Figure 3.18.png|400]]
- *comv_*: 조건에 따라 source에서 destination 레지스터로 이동
	- Source operand: 레지스터, 메모리
	- Destination operand: 레지스터
	- 두 operand 모두 16, 32, 64-bit 길이를 가짐(single-byte는 지원하지 않음)

#### Conditional move vs Conditional branch
- conditional branch는 프로세서가 분기 조건에 대한 계산이 끝나기 전까지 어느 branch로 갈지 결정할 수 없음
- 이를 해결하기 위해 프로세서는 branch prediction logic을 사용해 각 점프 명령이 어떤 방향으로 실행될지 추측함
- 분기 예측이 정확하면 pipeline은 인스트럭션들로 가득 채워지지만, 예측이 틀리면 이미 실행한 인스트럭션들을 폐기하고 다시 pipeline을 채워야 함(약 15~30 clock cycle의 성능 손실)
- conditional move를 사용하면 분기 예측에 의존하지 않아, 고정된 clock cycle을 필요로 하여 더 효율적인 pipelining 수행 가능

``` c
// general form of conditional expression
v = test-expr ? then-expr : else-expr;

// The standard way to compile this expression using conditional control
if (!test-expr)
	goto false;
v = then-expr;
	goto done;
false:
	v = else-expr;
done:

// using condtional move 
v = then-expr;
ve = else-expr;
t = test-expr;
if (!t) v = ve;
```
- conditional move 기반 코드는 `then-expr`과 `else-expr`이 모두 계산된 후 `test-expr`의 계산 결과에 따라 최종 jump 목적지가 결정됨
#### Bad Cases for Conditional Move
- 조건부 이동이 항상 효울적인 것은 아님
1.  Expensive Computations: `val = Test(x) ? Hard1(x) : Hard2(x);`
	- 복잡한 `Hard1(x)`와 `Hard2(x)` 두 코드가 모두 수행되어 계산 비용이 많이 듦
	- 계산이 매우 간단한 경우에만 효율적임
2. Risky Computations: `val = p ? *p : 0;`
	- `p`가 `null`인 경우에도 먼저 포인터 `p`가 가리키는 값 `*p`를 역참조 하려고함
	- null pointer dereferencing error 발생 위험 존재
``` c
long cread(long *xp) {
	return (xp ? *xp : 0);
}
```
``` sh
# long cread(long *xp)
# Invalid implementation of function cread
# xp in register %rdi
cread:
  movq (%rdi), %rax 	# v = *xp
  testq %rdi, %rdi 		# Test x
  movl $0, %edx 		# Set ve = 0
  cmove %rdx, %rax 		# If x==0, v = ve
  ret Return v
```
- `xp`가 `null`이여도  `movq (%rdi), %rax`을 먼저 수행해 null 포인터 역참조 오류가 발생
3.  Computations with side effects: `val = x > 0 ? x*=7 : x+=3;`
	- `x*=7`과 `x+=3`이 `x`의 값을 바꾸는 side effect 발생
	- `then-expr`과 `else-expr`이 `test-expr`에 값을 바꾸어 side effect를 발생시킴

### 3.6.7 Loops
C에서 `do-while`, `while`, `for`에 해당하는 직접적인 machine code 인스트럭션은 없지만, conditional test/jump를 함께 사용해서 반복문을 구현함

####  *Do-While Loops*
``` c
// general form
do
	body-statement
	while (test-expr);

// translate into goto statement
loop:
	body-statement
	t = test-expr;
if (t)
	goto loop;
```
- `body-statement`가 적어도 한 번은 실행됨

``` c
// C code
long fact_do(long n) {
	long result = 1;
	do {
		result *= n;
		n = n-1;
	} while (n > 1);
	return result;
}

// Equivalent goto version
long fact_do_goto(long n) {
	long result = 1;
loop:
	result *= n;
	n = n-1;
	if (n > 1)
		goto loop;
	return result;
}
```
``` sh
# Corresponding assembly-language code
# long fact_do(long n)
# n in %rdi
fact_do:
  movl $1, %eax 	# Set result = 1
.L2: 		# loop:
  imulq %rdi, %rax 	# Compute result *= n
  subq $1, %rdi 	# Decrement n
  cmpq $1, %rdi 	# Compare n:1
  jg .L2 		# If >, goto loop
  rep; ret 		# Return
```
-  factorial 프로그램의 do-while 버전

#### *While Loops*
``` c
// general form
while (test-expr)
	body-statement
```
- `test-expr`을 먼저 계산해서 `body-statement`를 실행하지 않을 수 있음
- 기계어로 번연하는 여러가지 중 두 가지 방법이 GCC에서 이용됨

1. jump to middle
``` c
// translate into goto statement
	goto test;
loop:
	body-statement
test:
	t = test-expr;
	if (t)
		goto loop;
```
- 루프의 맨 마지막에서 unconditional jump를 위한 초기 test를 수행

``` c
// C code
long fact_while(long n) {
	long result = 1;
	while (n > 1) {
		result *= n;
		n = n-1;
	}
	return result;
}

// Equivalent goto version
long fact_while_jm_goto(long n) {
	long result = 1;
	goto test;
loop:
	result *= n;
	n = n-1;
test:
	if (n > 1)
		goto loop;
	return result;
}
```
``` sh
# Corresponding assembly-language code
# long fact_while(long n)
# n in %rdi
fact_while:
  movl $1, %eax 	# Set result = 1
  jmp .L5 		# Goto test
.L6: 		# loop:
  imulq %rdi, %rax 	# Compute result *= n
  subq $1, %rdi 	# Decrement n
.L5: 		# test:
  cmpq $1, %rdi 	# Compare n:1
  jg .L6 		# If >, goto loop
  rep; ret 		# Return
```
-  jump to middle을 이용한 factorial 프로그램의 while 버전

2. guarded do
``` c
// translate into do-while statement
t = test-expr;
if (!t)
	goto done;
do
	body-statement
	while (test-expr);
done:

// translate into goto statement
t = test-expr;
if (!t)
	goto done;
```
- 초기 test가 실패할 경우 루프를 건너뛰도록 conditonal branch를 이용해 do-while로 번역
- do-while을 다시 goto로 변역

``` c
// C code
long fact_while(long n) {
	long result = 1;
	while (n > 1) {
		result *= n;
		n = n-1;
	}
	return result;
}

// Equivalent goto version
long fact_while_gd_goto(long n) {
	long result = 1;
	if (n <= 1)
		goto done;
loop:
	result *= n;
	n = n-1;
	if (n != 1)
		goto loop;
done:
	return result;
}
```
``` sh
# Corresponding assembly-language code
# long fact_while(long n)
# n in %rdi
fact_while:
  cmpq $1, %rdi 	# Compare n:1
  jle .L7 		# If <=, goto done
  movl $1, %eax 	# Set result = 1
.L6: 		# loop:
  imulq %rdi, %rax 	# Compute result *= n
  subq $1, %rdi 	# Decrement n
  cmpq $1, %rdi 	# Compare n:1
  jne .L6 		# If !=, goto loop
  rep; ret 		# Return
.L7: 		# done:
  movl $1, %eax 	# Compute result = 1
  ret 			# Return
```
- guarded-do를 이용한 factorial 프로그램의 while 버전
- `if (n != 1)`인 이유는 루프가 n > 1 일 때만 진입 가능해 	`if (n > 1)`과 동일함

#### *For Loops*
``` c
// general form
for (init-expr; test-expr; update-expr)
	body-statement

// translate into while statement
init-expr;
while (test-expr) {
	body-statement
	update-expr;
}
```
- 한 가지 예외를 제외하고 while 반복문과 동일하게 동작

``` c
// using jump to middle
	init-expr;
	goto test;
loop:
	body-statement
	update-expr;
test:
	t = test-expr;
	if (t)
		goto loop;

// using guarded do
init-expr;
	t = test-expr;
	if (!t)
		goto done;
loop:
	body-statement
	update-expr;
	t = test-expr;
	if (t)
		goto loop;
done:
```
- 최적화 수준에 따라 for에서 번역된 while문을  jump to middle, guarded do를 선택하여 goto로 번역

``` c
long fact_for(long n) {
	long i;
	long result = 1;
	for (i = 2; i <= n; i++)
		result *= i;
	return result;
}
```
- factorial 프로그램의 for 버전
- for문의 구성요소
	- init-expr:  i = 2
	- test-expr:  i <= n
	- update-expr:  i++
	- body-statement:  result \*= i;

``` c
// translate for loop into while statement
long fact_for_while(long n) {
	long i = 2;
	long result = 1;
	while (i <= n) {
		result *= i;
		i++;
	}
	return result;
}

// translate while loop into goto statement
long fact_for_jm_goto(long n) {
	long i = 2;
	long result = 1;
	goto test;
loop:
	result *= i;
	i++;
test:
	if (i <= n)
		goto loop;
	return result;
}
```
- for문을 이용한 factorial 함수를 while문으로, jump to middle 방법을 이용해 goto문으로 번역한 코드

``` sh
# long fact_for(long n)
# n in %rdi
fact_for:
  movl $1, %eax 	# Set result = 1
  movl $2, %edx 	# Set i = 2
  jmp .L8 			# Goto test
.L9: 		# loop:
  imulq %rdx, %rax 	# Compute result *= i
  addq $1, %rdx 	# Increment i
.L8: 		# test:
  cmpq %rdi, %rdx 	# Compare i:n
  jle .L9 			# If <=, goto loop
  rep; ret 			# Return
```
- GCC로 `-Og` 옵션을 이용해 생성한 어셈블리 코드

### 3.6.8 Switch Statements (p225 오타 개많, 원서 269 참고)
- *jump table*: 프로그램이 실행해야 하는 동작이 위치한 주소를 원소로 갖는 배열

``` c
// switch statement
void switch_eg(long x, long n, long *dest) {
	long val = x;
	switch (n) {
	case 100:
		val *= 13;
		break;
	case 102:
		val += 10;
		/* Fall through */
	case 103:
		val += 11;
			break;
	case 104:
	case 106:
		val *= val;
		break;
	default:
		val = 0;
	}
	*dest = val;
}
```
- GCC는 case 값의 밀집도와 case 수를 고려해서 switch문 번역 방법을 선택(jump table, conditional branch)
- 여러 개의 case가 있고, case 값들 사의 간격이 작다면 jump table 사용

``` c
// translate into extended C
void switch_eg_impl(long x, long n, long *dest) {
	/* Table of code pointers */
	static void *jt[7] = {
		&&loc_A, &&loc_def, &&loc_B,
		&&loc_C, &&loc_D, &&loc_def,
		&&loc_D
	};
	unsigned long index = n - 100;
	long val;

	if (index > 6)
		 goto loc_def;
	/* Multiway branch */
	goto *jt[index];

loc_A: /* Case 100 */
	val = x * 13;
	goto done;
loc_B: /* Case 102 */
	x = x + 10;
	/* Fall through */
loc_C: /* Case 103 */
	val = x + 11;
	goto done;
loc_D: /* Cases 104, 106 */
	val = x * x;
	goto done;
loc_def: /* Default case */
	val = 0;
done:
	*dest = val;
}
```
- 배열 `jt`에 각 레이블 주소를 원소로 저장
- `n - 100`을 해서 범위를 0~6 사이로 만들어 unsigned long 타입 `index`에 저장
- `index`가 0~6 범위에 벗어나는 지를 6을 초과하는지 시험하는 방식으로 수행(unsigned)
- `index`가 6을 초과하면 goto문을 이용해 `loc_def`(Default case)로 점프

``` sh
# void switch_eg(long x, long n, long *dest)
# x in %rdi, n in %rsi, dest in %rdx
switch_eg:
  subq $100, %rsi	 	# Compute index = n-100
  cmpq $6, %rsi 		# Compare index:6
  ja .L8 			# If >, goto loc_def
  jmp *.L4(,%rsi,8) 		# Goto *jg[index]
.L3: 			# loc_A:
  leaq (%rdi,%rdi,2), %rax 	# 3*x
  leaq (%rdi,%rax,4), %rdi 	# val = 13*x
  jmp .L2 			# Goto done
.L5: 			# loc_B:
  addq $10, %rdi 		# x = x + 10
.L6: 			# loc_C:
  addq $11, %rdi 		# val = x + 11
  jmp .L2 			# Goto done
.L7: 			# loc_D:
  imulq %rdi, %rdi 		# val = x * x
  jmp .L2 			# Goto done
.L8: 			# loc_def:
  movl $0, %edi 		# val = 0
.L2: 			# done:
  movq %rdi, (%rdx) 		# *dest = val
  ret 				# Return

# jump tables
  .section	.rodata
  .align 		# 8 Align address to multiple of 8
.L4:
  .quad .L3 		# Case 100: loc_A
  .quad .L8 		# Case 101: loc_def
  .quad .L5 		# Case 102: loc_B
  .quad .L6 		# Case 103: loc_C
  .quad .L7 		# Case 104: loc_D
  .quad .L8 		# Case 105: loc_def
  .quad .L7 		# Case 106: loc_D
```
- `jmp *.L4(,%rsi,8)` 
	- .L4에 jump table 시작 주소, `%rsi`에는 인덱스가 저장되어 있음
	- 점프 테이블의 각 주소가 8바이트이므로, `%rsi`에 8을 곱함
	- `*`: 주소로 간접 점프(indirect jump)
- .L5(Case 102):
	- `break` 문이 없어 goto문이 없음(jump하지 않음)
	- 다음 코드 블록까지 연속해서 실행

---