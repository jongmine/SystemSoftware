## 3.8 Array Allocation and Access
- C 언어에서 배열 내의 요소에 대한 포인터를 생성하고, 이러한 포인터로 산술 연산을 수행할 수 있음
- 최적화 컴파일러는 배열 인덱싱에 사용되는 주소 계산을 단순화하는데, 이로 인해 C 코드와 머신 코드로 번역되는 사이의 대응 관계를 해석하는 것이 어려워질 수 있음

### 3.8.1 Basic Principles
- Data type *T*, 정수형 상수 *N*에 대해 `T A[N]` 배열 선언
	- *L*  \* *N* 바이트의 연속적인 공간을 메모리에 할당(*L*은 자료형 *T*의 크기)
	- 식별자 `A`를 배열이 시작하는 위치(*x*A)를 가리키는 포인터로 사용(포인터 값은 *x*A)
	- 배열의 원소 *i*는 주소 *x*A + *L* \* *i*에 저장(원소의 범위는 0~*N*-1)
- 정수형 데이터 값들의 배열  `E`
	- `E`의 주소는 %rdx, `i`는 %rcx에 저장
	- `movl (%rdx,%rcx,4),%eax`: `E[i]`의 주소 계산(*x*E + 4*i*)을 수행하고 메모리 위치를 읽어 %eax에 저장
#### Example
``` c
char A[12];
char *B[8];
int C[6];
double *D[5];
```
- 각각 다음 표와 같은 크기를 가짐

| Arrray | Element size | Total size | Element *i* |
|---|---|---|---|
|  A   | 1  | 12| *x*A + *i*       |
|  B   | 8  | 64| *x*B + 8*i*      |
|  C   | 4  | 24| *x*C + 4*i*      |
|  D   | 8  | 40| *x*D + 8*i*      |

### 3.8.2 Pointer Arithmetic
- C는 포인터에 대한 산술을 허용하며, 계산된 값은 포인터가 참조하는 자료형의 크기에 따라 조정
- 자료형 *T*의 데이터를 가리키는 포인터 *p*의 값이 *x*p
	- `p+i`: *x*p + *L* * i (*L*은 자료형 *T*의 크기)
- 단항 연산자(unary) '&'와 '\*'는 포인터의 생성/역참조 수행 
	- `Expr`이 어떤 객체를 나타낼 때, `&Expr`은 객체의 주소를 나타냄
	- `AExpr`이 주소를 나타낼 때, `*AExpr`은 해당 주소에 위치한 값을 나타냄
	- 식 `Expr`과 `*&Expr`은 동일
- array subscripting 연산은 배열과 포인터에 모두 적용 가능 
	- 배열 참조 `A[i]`는 식 `*(A+i)`와 동일(*i*번째 배열 원소의 주소를 계산해서 이 메모리 주소에 접근)

#### Example
- 정수 배열 `E`의 시작 주소가 %rdx, 인덱스 i가 %rcx에 저장되어 있다고 가정

|Expression|Type|Value|Assembly code|
|---|---|---|---|
|`E` |int \*| *x*E| `movl %rdx,%rax`|
|`E[0]`| int| M\[*x*E\] |`movl (%rdx),%eax`|
|`E[i]`| int |M\[*x*E+ 4*i*\]| `movl (%rdx,%rcx,4),%eax`|
|`&E[2]`| int \*| *x*E+ 8 |`leaq 8(%rdx),%rax`|
|`E+i-1`| int \*| *x*E+ 4*i* − 4|`leaq -4(%rdx,%rcx,4),%rax`|
|`*(E+i-3)`| int |M\[*x*E+ 4*i* − 12\]| `movl -12(%rdx,%rcx,4),%eax`|
|`&E[i]-E`| long| *i*| `movq %rcx,%rax`|
- `E`와 관련된 몇 가지 표현식
- 배열 값을 리턴하는 연산들이 `int`형이므로, 4바이트 연산을 수행(movl, %eax)
- 포인터를 리턴하는 연산들이 `int *`형이므로, 8바이트 연산을 수행(movq, leaq, %rax)
	- `E+i-1`: `E`의 *i*-1번째 원소의 주소
	- `*(E+i-3)`: `E`의 *i*-3번째 원소의 값
- 동일한 자료 구조 내에서 두 포인터의 차를 계산할 수 있음
	- `&E[i]-E`: 두 주소의 차이를 자료형의 크기로 나눈 값

#### Note
1. Out of range behavior implementation-dependent. 
	- 배열의 범위를 벗어나는 동작의 결과는 프로그램 구현에 따라 다름. 
	- Ex) 크기가 5인 배열의 6번째 인덱스에 접근하려고 하면, 어떤 일이 일어날지는 프로그래밍 언어와 컴파일러, 운영 체제 등에 따라 다름. 일부 시스템에서는 오류를 발생시키고, 일부 시스템에서는 예상치 못한 결과를 리턴할 수 있음
2. No guaranteed relative allocation of different arrays. 
	- 서로 다른 배열이 메모리에서 상대적으로 어떻게 할당되는지에 대한 보장이 없음
	- Ex) 두 배열이 메모리에서 어떤 위치에 할당될지, 그리고 두 배열 사이에 어떤 간격이 있을지는 프로그램이나 시스템에 따라 다름.
### 3.8.3 Nested Arrays
- 배열의 할당과 참조에 대한 일반적인 원칙은 배열의 배열을 생성할 때도 동일하게 적용
- 배열의 원소들은 메모리에 *row major* 순서로 저장

``` c
int A[5][3];

typedef int row3_t[3];
row3_t A[5]
```
- `int A[5][3];`은 `typedef int row3_t[3]; row3_t A[5];`와 동일함
- 자료형 `row3_t`는 3개의 정수로 구성된 배열로 정의되며, 배열 `A`는 이러한 배열  5개를 원소로 가짐
- 배열은  3개의 정수를 저장하기 위해 12바이트 필요,  전체 배열 크기는 4 \* 5 \* 3 = 60바이트
- 배열 `A`는 5개의 행과 3개의 열을 갖는 이차원 배열로 볼 수 있으며, `A[0][0]`부터 `A[4][2]`까지 참조 가능
	![[Figure 3.36.png|150]]

#### 다차원 배열의 원소 접근
- 컴파일러가 원하는 배열 원소의 위치를 찾기 위해 해당 원소의 오프셋(인덱스에 따른 위치 차이)를 계산하는 코드를 생성
- 배열의 시작을 기본 주소로, 오프셋(possibly scaled)을 인덱스로 하는 MOV 인스트럭션 사용
- 배열 `T D[R][C];` (*L*: 자료형 *T*의 크기)
	- 원소 `D[i][j]`는 메모리 주소 `&D[i][j]`, 즉 *x*D+ *L*(*C* \* *i* + *j*)에 위치

``` sh
# A in %rdi, i in %rsi, and j in %rdx
[1] leaq (%rsi,%rsi,2), %rax 		# Compute 3i
[2] leaq (%rdi,%rax,4), %rax 		# Compute xA + 12i
[3] movl (%rax,%rdx,4), %eax 		# Read from M[xA + 12i + 4]
```
- `A[i][j]`를 레지스터 %eax에 복사하는 코드
- *x*A, *i*, *j*가 각각 레지스터 %rdi, %rsi, %rdx에 있다고 가정
- x86-64 주소연산의 덧셈/곱셈 기능을 사용하여 원소의 주소를 *x*A+ 12*i* + 4*j* = *x*A + 4(3*i* + *j*)로 계산

### 3.8.4 Fixed-Size Arrays
- C 컴파일러는 고정 크기의 다차원 배열에서 작동하는 코드에 대해 다양한 최적화 수행 가능
- 인덱스 계산에서 곱셈 연산을 회피할 수 있음
 
``` c
#define N 16
typedef int fix_matrix[N][N];

/* Compute i,k of fixed matrix product */
int fix_prod_ele (fix_matrix A, fix_matrix B, long i, long k) {
	long j;
	int result = 0;
	for (j = 0; j < N; j++)
		result += A[i][j] * B[j][k];
	return result;
}
```
- 배열 A와 B의 곱인 배열의 원소 *i*, *k*를 계산하는 코드
- A의 i행과 B의 k열의 내적(Σ (0≤*j*<*N*) a_*i*,*j* * b_*j*,*k*)

``` c
/* Compute i,k of fixed matrix product */
int fix_prod_ele_opt(fix_matrix A, fix_matrix B, long i, long k) {
	int *Aptr = &A[i][0];        /* Points to elements in row i of A */
	int *Bptr = &B[0][k];        /* Points to elements in column k of B */
	int *Bend = &B[N][k];        /* Marks stopping point for Bptr */
	int result = 0;
	do {                         /* No need for initial test */
		result += *Aptr * *Bptr; /* Add next product to sum */
		Aptr ++;                 /* Move Aptr to next column */
		Bptr += N;               /* Move Bptr to next row */
	} while (Bptr != Bend);      /* Test for stopping point */
	return result;
}
```
- 최적화된 코드 (gcc -O1 최적화 레벨)
1. 정수 인덱스 j를 제거하고 모든 배열 참조를 포인터 역참조로 변환 
	- `Aptr`: A의 *i*행의 연속하는 원소를 가리키는 포인터(초기값: `A`의 row *i*의 첫 번째 원소의 주소)
	- `Bptr`: B의 *k*열의 연속하는 원원소를 가리키는 포인터(초기값: `B`의 column *k*의 첫 번째 원소의 주소)
	- `Bend`: 루프를 종료할 때 `Bptr`이 갖는 주소 값(초기값: `B`의  column *j*의 (n + 1)번째 원소의 주소)    
2. do-while 루프에서 `Aptr`과 `Bptr`이 가리키는 요소들을 곱하여 `result`에 더함
3. `Aptr`과 `Bptr`은 각각 다음 열과 다음 행으로 이동
4. `Bptr`이 *k*열의 마지막 원소를 가리키게 되면 루프 종료

``` sh
# int fix_prod_ele_opt(fix_matrix A, fix_matrix B, long i, long k)
# A in %rdi, B in %rsi, i in %rdx, k in %rcx
[01] fix_prod_ele:
[02]   salq $6, %rdx 		    # Compute 64 * i
[03]   addq %rdx, %rdi 		    # Compute Aptr = xA + 64i = &A[i][0]
[04]   leaq (%rsi,%rcx,4), %rcx	# Compute Bptr = xB + 4k = &B[0][k]
[05]   leaq 1024(%rcx), %rsi    # Compute Bend = xB + 4k + 1024 = &B[N][k]
[06]   movl $0, %eax 		    # Set result = 0
[07] .L7: 			            # loop:
[08]   movl (%rdi), %edx 		# Read *Aptr
[09]   imull (%rcx), %edx 		# Multiply by *Bptr
[10]   addl %edx, %eax 		    # Add to result
[11]   addq $4, %rdi            # Increment Aptr ++
[12]   addq $64, %rcx           # Increment Bptr += N
[13]   cmpq %rsi, %rcx          # Compare Bptr:Bend
[14]   jne .L7                  # If !=, goto loop
[15]   rep; ret                 # Return
```
- `fix_prod_ele` 함수를 변환한 어셈블리 코드
- `&A[i][0]`: *x*A + 4(16 \* *i* + 0) = *x*A + 64*i* -> *i*를 왼쪽으로 6번 시프트해서 수행
- `&B[0][k]`: *x*B + 4(16 \* 0 + *k*) = *x*B + 4*k*

### 3.8.5 Variable-Size Arrays
- ISO C99 표준에서 배열의 차원을 결정하는 식이 런타임에 계산될 수 있게됨(배열의 크기가 컴파일 시점이 아닌 런타임에 결정될 수 있음)
- 배열을 선언할 때 식을 사용하여 배열의 차원을 지정할 수 있음(`int A[expr1][expr2]`)
- 가변크기 배열은 함수의 로컬 변수로, 함수의 인자로 사용할 수 있음(배열의 선언부에서 식 `expr1`과 `expr2`를 계산하여 배열의 차원을 결정)

``` c
int var_ele(long n, int A[n][n], long i, long j) {
	return A[i][j];
}
```
- *n* × *n* 배열의 *i*, *j* 원소를 리턴하는 함수
- *n*은 반드시 파라미터 `A[n][n]`보다 먼저 나와있어야 함

``` sh
# int var_ele(long n, int A[n][n], long i, long j)
# n in %rdi, A in %rsi, i in %rdx, j in %rcx
[1] var_ele:
[2]   imulq %rdx, %rdi 		    # Compute n * i
[3]   leaq (%rsi,%rdi,4), %rax 	# Compute xA + 4(n * i)
[4]   movl (%rax,%rcx,4), %eax 	# Read from M[xA + 4(n * i) + 4j ]
[5]   ret
```
- 원소 *i*, *j*의 주소를 *x*A + 4(*n* \* *i*) + 4*j* = *x*A + 4(*n* \* *i* + *j*)로 계산
- *n* \* *i*를 계산하기 위해 `leaq` 대신 `imulq` 사용
- 비트 시프트와 더하기 연산 대신 곱셈을 사용해야 하는 경우가 있음(성능 저하 발생 가능)

``` c
/* Compute i,k of variable matrix product */
int var_prod_ele(long n, int A[n][n], int B[n][n], long i, long k) {
	long j;
	int result = 0;

	for (j = 0; j < n; j++)
		result += A[i][j] * B[j][k];

	 return result;
 }
```
- 가변 크기 배열을 이용한 행렬의 곱에서 *i*, *k* 원소를 계산하는 코드
- 배열 A와 B의 곱인 배열의 요소 *i*, *k*를 계산하는 코드
- A의 i행과 B의 k열의 내적(Σ (0≤*j*<*N*) a_*i*,*j* * b_*j*,*k*)

``` c
/* Compute i,k of variable matrix product */
int var_prod_ele_opt(long n, int A[n][n], int B[n][n], long i, long k) {
	int *Arow = A[i];
	int *Bptr = &B[0][k];
	int result = 0;
	long j;
	for (j = 0; j < n; j++) {
		result += Arow[j] * *Bptr;
		Bptr += n;
	}
	return result;
}
```
- 배열 원소에 접근 효율을 높이기 위해 최적화된 코드(가변크기 배열의 규칙적인 접근 패턴을 이용)
1. 인덱스 계산의 복잡성을 줄이기 위해 포인터를 사용하여 배열 접근
	- `Arow`: 행렬 A의 *i*번째 행의 주소를 가리키는 포인터
	- `Bptr`: 행렬 B의 *k*번째 열의 주소를 가리키는 포인터
2. 기존 코드에서 발생하던 '*n* \* *i* + *j*'와 같은 복잡한 인덱스 계산 회피
	- 루프 내에서 `Bptr` 포인터는 `n`만큼 증가하여 행렬 B의 다음 행으로 이동
	- `Arow[j]`는 배열 인덱스 `j`를 이용하여 행렬 A의 다음 열로 이동
	- `Arow[j]`는 `A[i][j]`를, `*Bptr`는 `B[j][k]`를 나타냄

``` sh
# Registers: n in %rdi, Arow in %rsi, Bptr in %rcx
# 4n in %r9, result in %eax, j in %edx
[1] .L24:			 		    # loop:
[2]   movl (%rsi,%rdx,4), %r8d 	# Read Arow[j]
[3]   imull (%rcx), %r8d 		# Multiply by *Bptr
[4]   addl %r8d, %eax 		    # Add to result
[5]   addq $1, %rdx 		    # j++
[6]   addq %r9, %rcx 		    # Bptr += n
[7]   cmpq %rdi, %rdx 		    # Compare j:n
[8]   jne .L24 				    # If !=, goto loop
```
- `var_prod_ele_opt` 함수를 변환한 어셈블리 코드 중 루프 부분
- *%r9*: 다음 행으로 이동시키기 위해 더하는 값으로, *n*에 정수형 크기 4를 곱함
- `movl (%rsi,%rdx,4), %r8d`: `Arow[j]`의 주소로, `Arow`의 시작 주소(`%rsi`)에 `j*4`를 더하여 `Arow[j]`의 주소를 계산(*j*에 정수형 크기 4를 곱함)
- `cmpq %rdi, %rdx`: `j`와 `n`을 비교하여 같지 않으면, `jne .L24`에서 다시 루프를 진행시킴 

#### Note(Dynamic Nested Array)
1. Must do index computation explicitly.
	- 각 원소에 접근하기 위해 인덱스 계산을 명시적으로 수행해야 함
	- Ex) 2차원 배열 `arr[i][j]`에서 특정 원소에 접근하려면, `i`와 `j` 인덱스를 각각 계산
2. Accessing single element costly.
	- 단일 원소에 접근하는 것은 비용이 많이 듦(여러 단계의 메모리 접근이 필요할 수 있음)
3. Must do multiplication.
	- 인덱스 계산에 곱셈 연산이 필요
	 - Ex) `arr[i][j]`의 주소를 계산하려면 `i`번째 행의 크기에 `j`를 곱해야 함

---
## 3.9 Heterogeneous Data Structures
- C에서 서로 다른 타입의 객체를 결합하여 데이터 타입을 생성할 수 있음(struct, union)

### 3.9.1 Structures
- 서로 다른 유형의 객체들을 하나의 객체로 묶어주는 자료형
- 하나의 구조체 내의 서로 다른 컴포넌트들은 이름으로 참조됨
- 모든 컴포넌트들은 메모리의 연속된 영역에 저장
- 포인터는 첫 번째 바이트의 주소(배열과 유사)
- 컴파일러는 각 필드의 바이트 오프셋을 가리키는 각 구조체 타입에 대한 정보를 관리, 이 오프셋을 이용해 구조체 원소에 대한 참조를 생성
	- *byte offset*: 구조체의 시작점으로부터 각 필드가 위치한 곳까지의 바이트 단위 거리
- 컴파일러는 각 필드에 접근하기 위해 구조체 주소에 적절한 오프셋 값을 더하는 코드 생성

``` c
struct rec {
	int i;
	int j;
	int a[2];
	int *p;
};

/*
Offset  |0    |4    |8            |16   |24 
Contens	|--i--|--j--|-a[0]-|-a[1]-|--p--|
*/
```
- 정수형(4바이트) 변수 2개, 2개의 정수로 이루어진 배열(8바이트) 1개, 정수형 포인터(8바이트) 2개 -> 총 24바이트

``` sh
# Registers: r in %rdi
movl (%rdi), %eax 		# Get r->i
movl %eax, 4(%rdi) 		# Store in r->j
```
- `r->i` 필드를 `r->j`로 복사하는 어셈블리 코드
- `r`의 주소에 오프셋 4 를 더함

``` sh
# Registers: r in %rdi, i %rsi
leaq 8(%rdi,%rsi,4), %rax 		# Set %rax to &r->a[i]
```
- `&(r->a[i])`에 접근하는 어셈블리 코드
- 오프셋: 4(필드 `i`) + 4(필드 `j`) + 4(필드 `a[2]`의 원소 자료형 크기)\ * 1(*i*번째 인덱스) = 8 + 4*i*

``` sh
# Registers: r in %rdi
movl 4(%rdi), %eax 		    # Get r->j
addl (%rdi), %eax 		    # Add r->i
cltq 				    	# Extend to 8 bytes
leaq 8(%rdi,%rax,4), %rax 	# Compute &r->a[r->i + r->j]
movq %rax, 16(%rdi) 		# Store in r->p
```
- `r->p = &r->a[r->i + r->j]`에 해당하는 어셈블리 코드
- %rdi에 8바이트 오프셋을 더하여 배열 `a[]`에 접근 후, 4바이트(정수형 크기) \* (`i`+`j`) 오프셋을 더하여 접근
- `r->p`는 이전 필드의 모든 오프셋 16을 더하여 접근

### 3.9.2 Unions
- 여러 가지 타입의 데이터를 한 공간에 저장할 수 있음
- 모든 필드가 동일한 메모리 블록을 참조(모든 필드가 동일한 메모리 공간을 공유)
- 한 번에 한 가지 타입의 데이터만 저장할 수 있음(마지막으로 저장한 값만 유지)
- 여러 필드가 동일한 메모리 공간을 공유하므로, 한 필드에 데이터를 저장하고 다른 필드를 통해 그 데이터를 읽을 수 있음

### 3.9.3 Data Alignment
- *alignment restrictions*: 기본 자료형에 대한 허용 가능한 주소에 제한을 두고 있음
	- 일부 객체의 주소가 특정 값 *K*(일반적으로 2, 4, 8)의 배수여야 함
	- 프로세서와 메모리 시스템 사이의 인터페이스를 구성하는 하드웨어 설계를 단순화함
	- 메모리 접근의 효율성을 높이고, 프로세서와 메모리 사이의 데이터 전송을 최적화
- 데이터를 정렬하는 이유
	- 메모리는 (정렬된) double-word 또는 quad-word로 접근함
	- quad word 경계를 넘어 데이터를 로드하거나 저장하는 것은 비효율적
	- 데이터가 2페이지를 넘어갈 때 가상 메모리를 처리하는 것은 매우 까다로움

|K|Type|
|---|---|
|1|char|
|2|short|
|4|int, float|
|8|long, double, char|
- x86-64 하드웨어는 데이터의 정렬과 상관없이 정확하게 작동하지만, Intel은 메모리 시스템의 성능 향상을 위해 데이터 정렬을 권장함
- 컴파일러는 전역 데이터에 대한 원하는 정렬을 지시하는 어셈블리 코드에 *directive*들을 삽입함(Ex. `.align 8`: 다음에 오는 데이터가 8배수인 주소로 시작)

#### structure alignment
- 컴파일러는 각 구조체 원소가 각각의 정렬 요구사항을 만족하도록, 필드 할당 시 gap을 삽입
``` c
struct S1 {
	int i;
	char c;
	int j;
};

/* not aligned
Offset  |0    |4|5    |9 
Contens	|--i--|c|--j--|
*/

/* algined structure
Offset  |0    |4|5 |8    |12 
Contens	|--i--|c|==|--j--|
*/
```
- 구조체 `S1`을 최소 크기인 9바이트 할당 시, 4바이트 정렬요건을 만족할 수 없음
- 컴파일러는 필드 `c`와 `j` 사이에 3바이트 gap을 삽입하여 정렬
	- 전체 구조체 크기: 12바이트
	- 모든 필드가 4바이트 정렬 요건을 만족

``` c
struct S2 {
	int i;
	int j;
	char c;
};

/* algined structure
Offset  |0    |4   |8 |9  |12
Contens	|--i--|--j--|c|===|
*/
```
- 구조체 `S2`을 최소 크기인 9바이트 할당 시, 필드 `i`, `j`에 대해 4바이트 정렬요건을 만족
	- `struct S2 d[4]`와 같이 구조체 배열 사용시, `d`의 각 원소들의 정렬 요건을 만족시킬 수 없음(원소들은각각  *x*d, *x*d + 9, x*d* + 18, x*d* + 27의 주소를 갖게 됨)
- 컴파일러는 구조체 `S2`에 마지막 3바이트는 padding으로 하여 12바이트를 할당
	- `d`의 원소들은 각각 *x*d, *x*d + 12, *x*d + 24, *x*d + 36의 주소를 갖게 되어, *x*d가 4의 배수라면 모든 정렬제한을 만족

---