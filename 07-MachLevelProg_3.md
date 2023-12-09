## 3.7 Procedures
#### Example 
프로시저 P가 프로시저 Q를 호출하고, 다시 P로 리턴할 때를 가정
- *제어권 전달(Passing control)*
	- 프로그램 카운터(PC)를 Q에 대한 코드가 시작주소로 설정
	- 리턴할 때 P에서 Q를 호출하는 인스트럭션의 다음 인스트럭션으로 설정
- *데이터 전달(Passing data)*
	- P는 하나 이상의 parameter를 Q에 전달 
	- Q는 하나의 리턴 값을 P로 다시 전달
- *메모리 할당/반납(Allocating and deallocating memory)* 
	- Q는 시작할 때 local variable들을 위한 공간을 할당
	- 리턴할 때 할당한 저장공간을 반납

### 3.7.1 The Run-Time Stack
![[Figure 3.25.png|250]]
- 프로그램은 스택을 사용해서 프로시저에 필요한 저장소를 관리함
- x86-64 스택은 낮은 주소 방향으로로 성장, 스택 포인터 *%rsp*는 스택의 맨 위 요소를 가리킴
- 프로시저가 레지스터가 저장할 수 있는 개수(인자 6개)를 넘어서 저장 공간을 필요로 할 때, 공간을 스택(프로시저의 스택 프레임(stack frame))에 할당  
- 프로시저 P가 Q를 호출할 때, 리턴 주소를 스택에 push하여 Q가 리턴된 후 P에서 실행을 재개할 위치를 가리킴
- 스택 프레임 대부분이 고정된 크기를 갖고 프로시저 시작 시 할당됨(일부 프로시저는 가변 크기 필요)
- 효율성을 위해 x86-64 프로시저들은 필요한 스택 프레임 부분만 할당
- *leaf procedure*: 모든 로컬 변수를 레지스터에 저장 가능, 다른 함수를 호출하지 않는 프로시저로 스택 프레임을 필요로 하지 않음

### 3.7.2 Control Transfer
- *call Label / *Operand*: Procedure call
	- Destination operand: 호출된 프로시저가 시작하는 인스트럭션의 주소로, Label 또는 간접 참조 오퍼랜드(\*Operand)를 가짐
	- 리턴 주소(call 인스트럭션의 바로 다음 인스트럭션)를 스택에 저장하고 동시에 호출된 프로시저로 점프
- *ret*: Return from call
	- 리턴할 주소를 스택에서 pop하여 프로그램 카운터(PC) 값으로 정한 후 이 주소로 점프
#### Example: Illustration of call and ret functions
``` sh
# Beginning of function multstore
[1] 0000000000400540 <multstore>:
[2] 400540: 53 				push %rbx
[3] 400541: 48 89 d3 		mov %rdx,%rbx
. . .
# Return from function multstore
[4] 40054d: c3 				retq
. . .
# Call to multstore from main
[5] 400563: e8 d8 ff ff ff 	callq 400540 <multstore>
[6] 400568: 48 8b 54 24 08 	mov 0x8(%rsp),%rdx
```
![[Figure 3.26.png|500]]
#### Example: Detailed execution of program involving procedure calls and returns
``` sh
# Disassembly of leaf(long y)
# y in %rdi
[01] 0000000000400540 <leaf>:
[02] 400540: 48 8d 47 02 		# lea 0x2(%rdi),%rax L1: z+2
[03] 400544: c3 			    # retq L2: Return
[04] 0000000000400545 <top>:
# Disassembly of top(long x)
# x in %rdi
[05] 400545: 48 83 ef 05 		# sub $0x5,%rdi T1: x-5
[06] 400549: e8 f2 ff ff ff 	# callq 400540 <leaf> T2: Call leaf(x-5)
[07] 40054e: 48 01 c0 		    # add %rax,%rax T3: Double result
[08] 400551: c3 			    # retq T4: Return
. . .
# Call to top from function main
[09] 40055b: e8 e5 ff ff ff 	# callq 400545 <top> M1: Call top(100)
[10] 400560: 48 89 c2 		    # mov %rax,%rdx M2: Resume
```
![[Figure 3.27.png|500]]
- 함수를 호출하면 기본적으로 스택 포인터(%rsp) 주소값이 8바이트씩 감소(x86-64에서 주소 크기)
- 리턴할 때마다 %rsp는 다시 이전 상태로 돌아오며, 최종적으로 원래 주소 값을 가짐
### 3.7.3 Data Transfer
- x86-64에서는 최대 6개의 정수형(정수, 포인터) 인자를 레지스터를 통해 전달 가능
- 레지스터들은 지정된 순서대로 사용
- 64-bit 보다 작은 인자들은 64-bit 레지스터의 일부분을 이용해서 접근 가능(Ex. 32-bit -> %edi로 접근)
- 함수가 6개 이상의 정수형 인자를 가질 경우, 나머지 인자들은 스택(argument build area)을 통해 전달
	- 인자 1~6은 레지스터에 복사, 7~n은 인자 7을 top으로 하여 스택에 저장
	- 스택을 통해 인자를 전달할 때, 모든 데이터 크기는 8배수로 반올림 
![[Figure 3.28.png|400]]
 
``` c
void proc(long a1, long *a1p,
		  int a2, int *a2p,
		  short a3, short *a3p,
		  char a4, char *a4p) {
	*a1p += a1;
	*a2p += a2;
	*a3p += a3;
	*a4p += a4;
}
```
- 각각 다른 크기의 정수(8, 4, 2, 1 바이트)들과 8바이트의 포인터 인자

``` sh
# void proc(a1, a1p, a2, a2p, a3, a3p, a4, a4p)
# Arguments passed as follows:
# a1 in %rdi (64 bits)
# a1p in %rsi (64 bits)
# a2 in %edx (32 bits)
# a2p in %rcx (64 bits)
# a3 in %r8w (16 bits)
# a3p in %r9 (64 bits)
# a4 at %rsp+8 ( 8 bits)
# a4p at %rsp+16 (64 bits)
[1] proc:
[2]   movq 16(%rsp), %rax 	    # Fetch a4p (64 bits)
[3]   addq %rdi, (%rsi) 		# *a1p += a1 (64 bits)
[4]   addl %edx, (%rcx) 		# *a2p += a2 (32 bits)
[5]   addw %r8w, (%r9) 		    # *a3p += a3 (16 bits)
[6]   movl 8(%rsp), %edx 		# Fetch a4 ( 8 bits)
[7]   addb %dl, (%rax) 		    # *a4p += a4 ( 8 bits)
[8]   ret 				        # Return
```
- 첫 6개 인자는 레지스터를 통해 전달

![[Figure 3.30.png|300]]
- 마지막 2개 인자는 스택을 통해 전달

### 3.7.4 Local Storage on the Stack
대부분 레지스터에 저장할 수 있는 범위를 넘어서 로컬 데이터 저장을 요구하지 않지만, 때로는 로컬 데이터를 메모리에 저장해야 하는 경우들이 있음
- 모든 로컬 데이터가 들어갈 충분한 레지스터가 없는 경우
- 주소 연산자 '&'가 로컬 변수에 적용되어 그에 대한 주소를 생성해야 하는 경우
- 로컬 변수 중 일부가 배열/구조체여서, 이를 참조를 통해 접근해야 하는 경우 

#### Example: Example of procedure definition and call
- Code for swap_add and calling function
``` c
long swap_add(long *xp, long *yp) {
	long x = *xp;
	long y = *yp;
	*xp = y;
	*yp = x;
	return x + y;
}

long caller() {
	long arg1 = 534;
	long arg2 = 1057;
	long sum = swap_add(&arg1, &arg2);
	long diff = arg1 - arg2;
	return sum * diff;
}
```

- Generated assembly code for calling function
``` sh
# long caller()
[01] caller:
[02] subq $16, %rsp 		# Allocate 16 bytes for stack frame
[03] movq $534, (%rsp) 		# Store 534 in arg1
[04] movq $1057, 8(%rsp) 	# Store 1057 in arg2
[05] leaq 8(%rsp), %rsi 	# Compute &arg2 as second argument
[06] movq %rsp, %rdi 		# Compute &arg1 as first argument
[07] call swap_add 		    # Call swap_add(&arg1, &arg2)
[08] movq (%rsp), %rdx 		# Get arg1
[09] subq 8(%rsp), %rdx 	# Compute diff = arg1 - arg2
[10] imulq %rdx, %rax 		# Compute sum * diff
[11] addq $16, %rsp 		# Deallocate stack frame
[12] ret 				    # Return
```
- 스택에 16바이트 할당하여 %rsi, %rdi 레지스터에 복사 후 swap
- 계산하여 리턴 후 다시 스택에서 16바이트 메모리 반납 

#### Example: Code to call function proc, defined in Figure 3.29
- C code for calling function
``` c
long call_proc() {
	long x1 = 1; int x2 = 2;
	short x3 = 3; char x4 = 4;
	proc(x1, &x1, x2, &x2, x3, &x3, x4, &x4);
	return (x1+x2)*(x3-x4);
}
```

- Generated assembly code
``` sh
# long call_proc()
[01] call_proc:
# Set up arguments to proc
[02] subq $32, %rsp 		# Allocate 32-byte stack frame
[03] movq $1, 24(%rsp) 		# Store 1 in &x1
[04] movl $2, 20(%rsp) 		# Store 2 in &x2
[05] movw $3, 18(%rsp) 		# Store 3 in &x3
[06] movb $4, 17(%rsp) 		# Store 4 in &x4
[07] leaq 17(%rsp), %rax 	# Create &x4
[08] movq %rax, 8(%rsp) 	# Store &x4 as argument 8
[09] movl $4, (%rsp) 		# Store 4 as argument 7
[10] leaq 18(%rsp), %r9 	# Pass &x3 as argument 6
[11] movl $3, %r8d 		    # Pass 3 as argument 5
[12] leaq 20(%rsp), %rcx 	# Pass &x2 as argument 4
[13] movl $2, %edx 		    # Pass 2 as argument 3
[14] leaq 24(%rsp), %rsi 	# Pass &x1 as argument 2
[15] movl $1, %edi 		    # Pass 1 as argument 1
# Call proc
[16] call proc
# Retrieve changes to memory
[17] movslq 20(%rsp), %rdx 	# Get x2 and convert to long
[18] addq 24(%rsp), %rdx 	# Compute x1+x2
[19] movswl 18(%rsp), %eax 	# Get x3 and convert to int
[20] movsbl 17(%rsp), %ecx 	# Get x4 and convert to int
[21] subl %ecx, %eax 		# Compute x3-x4
[22] cltq 		        	# Convert to long
[23] imulq %rdx, %rax 		# Compute (x1+x2) * (x3-x4)
[24] addq $32, %rsp 		# Deallocate stack frame
[25] ret 				    # Return
```
- 로컬 변수를 위해 스택에 저장공간을 할당, 8개의 인자를 갖는 함수
- 스택에 32바이트 할당하고 주소값을 저장하는 인자와 7~8번 인자를 스택에 저장
- 1~6번 인자를 레지스터에 복사하여 proc 함수 호출
- 모든 연산 후 다시 스택에서 32바이트 메모리 반납 
- \[7]번 라인에서 %rax는 단순히 임의로 저장하기 위해 사용됨
![[Figure 3.33.png|300]]

### 3.7.5 Local Storage in Registers
- 프로그램 레지스터가 모든 프로시저에 의해 공유되는 단일 리소스로 작동(모든 함수와 프로시저가 동일한 레지스터 set을 사용함)
- 한 프로시저가 다른 프로시저를 호출할 때, 호출된 프로시저가 호출하는 프로시저가 나중에 사용할 계획이었던 레지스터 값을 덮어쓰지 않아야 함
- 관습적으로 *%rbx*, *%rbp*, *%r12*~*%r15* 레지스터들은 callee-saved 레지스터로 분류
- 프로시저 P가 프로시저 Q를 호출할 때, Q는 callee-saved 레지스터의 값을 보존해야 함
	- 프로시저 Q는 이 값을 전혀 변경하지 않거나, 스택(saved registers 영역)에 push해두고 리턴하기 전에 pop해서 값을 보존
- 스택 포인터 *%rsp*를 제외한 모든 레지스터들은 caller-saved 레지스터로 분류(함수에 의해 변경 가능)

#### Example: Code demonstrating use of callee-saved registers
- Calling function
``` c
long P(long x, long y) {
	long u = Q(y);
	long v = Q(x);
	return u + v;
}
```

- Generated assembly code for the calling function
``` sh
# long P(long x, long y)
# x in %rdi, y in %rsi
[01] P:
[02] pushq 		        # %rbp Save %rbp
[03] pushq 		        # %rbx Save %rbx
[04] subq $8, %rsp 	    # Align stack frame
[05] movq %rdi, %rbp 	# Save x
[06] movq %rsi, %rdi 	# Move y to first argument
[07] call Q 		    # Call Q(y)
[08] movq %rax, %rbx 	# Save result
[09] movq %rbp, %rdi 	# Move x to first argument
[10] call Q 		    # Call Q(x)
[11] addq %rbx, %rax 	# Add saved Q(y) to Q(x)
[12] addq $8, %rsp 	    # Deallocate last part of stack
[13] popq %rbx 		    # Restore %rbx
[14] popq %rbp 		    # Restore %rbp
[15] ret
```
- %rbx, %rbp 레지스터를 사용하기 위해 기존 값을 스택에 저장 후 , 각각 x, Q(y) 값을 %rbx, %rbp에 저장
- 함수 Q를 호출할 때 필요한 인자를 %rbx, %rbp 레지스터에서 불러옴
- 모든 연산이 끝난 후 리턴하기 전에 스택에 저장되어 있던 기존 두 callee-saved 레지스터의 값을 pop해서 %rbx, %rbp에 복원
- x86-64에서 스택 프레임이 16바이트 경계에 정렬되어 있어야 함
	- \[7]번 라인에서 call 함으로써 8바이트 할당이 되므로, \[7]번 라인 이후에도 16바이트를 맞추기 위해 \[4]번 라인에서 스택에 8바이트 할당
	- \[12]번 라인에서 리턴하기 전에 할당했던 공간을 다시 반납 

### 3.7.6 Recursive Procedures
- 스택 관리 기법으로 여러 개의 호출이 서로 간섭하지 않도록 각 함수 호출마다 스택에 자신만의 private 공간(리턴 주소와 callee-saved 레지스터 기존 값)을 확보 
- 스택은 함수가 호출될 때 로컬 저장소를 할당하고 리턴하기 전에 할당을 해제(함수의 호출-리턴 순서와 자연스럽게 일치하는 할당/해제)
- 상호 재귀(Ex. 함수 P가 Q를 호출하고, Q가 다시 P를 호출하는 경우)와 같은 더 복잡한 패턴에 대해서도 작동함

#### Example: Code for recursive factorial program
- C code
``` c
long rfact(long n) {
	long result;
	if (n <= 1)
		result = 1;
	else
		result = n * rfact(n-1);
	return result;
}
```

- Generated assembly code
``` sh
# long rfact(long n)
# n in %rdi
[01] rfact:
[02]   pushq %rbx 		    # Save %rbx
[03]   movq %rdi, %rbx  	# Store n in callee-saved register
[04]   movl $1, %eax    	# Set return value = 1
[05]   cmpq $1, %rdi    	# Compare n:1
[06]   jle .L35 	   	    # If <=, goto done
[07]   leaq -1(%rdi), %rdi  # Compute n-1
[08]   call rfact 		    # Call rfact(n-1)
[09]   imulq %rbx, %rax 	# Multiply result by n
[10] .L35: done:
[11]   popq %rbx 	    	# Restore %rbx
[12]   ret 		            # Return

```
- 재귀적 팩토리얼 함수
- 먼저 스택에 *%rbx* 레지스터의 기존 값을 저장하고, 리턴하기 전에 값을 복원
- 재귀 호출 `rfact(n-1)`의 리턴 값은 `%rax` 레지스터에 저장되고, n의 값은 `%rbx` 레지스터에 저장되어 `n * rfact(n-1)`이 수행됨

---