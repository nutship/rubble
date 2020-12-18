### 准备

<font class="u_1">

-   只能用限制内的操作符，且有数量限制
    -   222
-   只能引用 `0x00-0xff` 内的常量
-   所有定义写在所有赋值前
-   `btest` 验证结果正确性，`dlc` 检查操作符数量和是否符合要求，`drivel.pl` 最终评分

</font>

### 1`

#### `bitxor`

只用 `~` 和 `&` 实现 `^`： `x^y = (~x & y) | (x & ~y) = ...`

```C
/* Legal ops: ~ &, Max ops: 14 */
int bitXor(int x, int y) {
    return ~(~(~x & y) & ~(x & ~y));
}
```

### 2`

#### `isTmax`

returns 1 if x is the maximum, two's complement number

```C
/* Legal ops: ! ~ & ^ | +,  Max ops: 10 */
int isTmax(int x) {
    // return !((x + 1) ^ ~x) & !!(x + 1);
    return !(x + 2 + x) & !!(x + 1);
}
```

!!! caution "&emsp;"

    如果改为 `return !(x + x + 2) & !!(x + 1);` 就会出错，问题代码:

    ```C
    void isTmax(int x) {
        int y = !(x + x + 2); // bug: y=0
        printf("%d\n", y);
    }

    int main() { isTmax(0x7fffffff); }
    ```
    只在 `gcc -O0` 的等级下会输出 1，具体原因不明

#### `allOddBits`

return 1 if all odd-numbered bits in word set to 1 where bits are numbered from 0 to 31

```C
/* Legal ops: ! ~ & ^ | + << >>,  Max ops: 12 */
int allOddBits(int x) {
    int aa = 0xAA;
    aa += aa << 8;
    aa += aa << 16;
    return !(~x & aa);
}
```

### 3`

#### `isAsciiDigit`

return 1 if `0x30 <= x <= 0x39`<br>
思路: 观察 `0x0 - 0x9` 比特位的特点，中间两个位不应该有值，除非 `x<8`

```C
/* Legal ops: ! ~ & ^ | + << >>,  Max ops: 15 */
int isAsciiDigit(int x) {
    int low4b = x & 0x0f;
    int h28b = x >> 4;
    return (!(h28b + ~0x02)) & (!(low4b & 0x06) | !(low4b >> 3));
}
```

#### `conditional`

same as `x ? y : z`

```C
/*  Legal ops: ! ~ & ^ | + << >>, Max ops: 16 */
int conditional(int x, int y, int z) {
    int mask1 = ~0, mask2 = 0;
    mask1 += !x, mask2 = ~mask1;
    return (y & mask1) + (z & mask2);
}
```

#### `isLessOrEqual`

if x <= y then return 1, else return 0

| `r=x-y `     | x+- | y+- | r+- | overflow | zero |
| :----------- | :-: | :-: | :-: | :------: | :--: |
| `x>y, x-y>0` |  ?  |  ?  |  1  |    0     |  0   |
| `x>y, x-y<0` |  1  |  0  |  0  |   1(+)   |  0   |
| `x=y`        |  ?  |  ?  |  0  |    0     |  1   |
| `x<y, x-y<0` |  ?  |  ?  |  0  |    0     |  0   |
| `x<y, x-y>0` |  0  |  1  |  1  |   1(-)   |  0   |

```C
/* Legal ops: ! ~ & ^ | + << >>,  Max ops: 24 */
int isLessOrEqual(int x, int y) {
    int xtag = (x >> 31) + 1;
    int ytag = (y >> 31) + 1;
    int res = x + ~y + 1;
    int rtag = ((res >> 31) + 1) & (!!res);
    // x>0, y<0, r<=0 (+) OR r<0, y>0, r>0 (-)
    int isOverflow = (xtag & (!ytag) & (!rtag)) | ((!xtag) & ytag & rtag);
    return !(isOverflow ^ rtag);
}
```

### 4`

#### `logicalNeg`

implement the `!` operator, using all of the legal operators except `!` <br>
将所有非零数映射到 0，利用 `f(x)=(x >> 31) | (-x >> 31)`，分为:

-   `f(0) = 0|0 = 0`
-   `f(0x80000000) = -1|-1 = -1`
-   `f(x) = f(-x) = 0|-1 = -1 `

```C
/* Max ops: 12,  Rating: 4 */
int logicalNeg(int x) {
    return (x >> 31 | (~x + 1) >> 31) + 1;
}
```

#### `howManyBits`

return the minimum number of bits required to represent x in two's complement, eg:

-   `howManyBits(0x80000000) = 32`
-   `howManyBits(0) = howManyBits(-1) = 1`

对于有符号数 $x$，$ -2^{n-1} \leq x \leq 2^{n-1}-1 $，那么问题转化为求 $n$ <br>

$$
\qquad n = f(x) =
\begin{cases}
   \lceil \log_2(x+1) \rceil  &\text{if } x \geq 0 \\\\
   \lceil \log_2(-x) \rceil&\text{if } x < 0
\end{cases}
$$

将 $x$ 取反，由 $f(x) = f(-x-1)$，结果不变，因此可通过补码将 $x$ 先统一表示为正/负数 (正数查左边的 0，负数查 1)。然后可以:

-   统一表示为正，然后二分: 如果左边有 1，右边的位全计入，向左递归；如果没有 1，计 0，向右递归
-   把一个正数视为多个小整数的拼接，然后边移位边统计

=== "bisection (34 ops)"

    ```C
    /* Legal ops: ! ~ & ^ | + << >>,  Max ops: 90 */
    int howManyBits(int x) {
        int lz1, lz2, lz3, lz4, lz5, lz6, tmp;
        // uniform rep as pos
        int rep = x ^ (x >> 31);

        tmp = !!(rep >> 16);  lz1 = tmp << 4;  rep >>= lz1;
        tmp = !!(rep >> 8);   lz2 = tmp << 3;  rep >>= lz2;
        tmp = !!(rep >> 4);   lz3 = tmp << 2;  rep >>= lz3;
        tmp = !!(rep >> 2);   lz4 = tmp << 1;  rep >>= lz4;
        tmp = !!(rep >> 1);   lz5 = tmp;       rep >>= lz5;
        lz6 = !!rep;

        return lz1 + lz2 + lz3 + lz4 + lz5 + lz6 + 1;
    }
    ```

=== "naive (56 ops)"

    ```C
    /* Legal ops: ! ~ & ^ | + << >>,  Max ops: 90 */
    int howManyBits(int x) {
        // uniform rep as neg
        int xorMask = ((x >> 31) & 1) + ~0;
        int rep = x ^ xorMask;
        int result = 0;

        // construct the mask
        int seg4 = rep >> 24;
        int seg3 = rep << 8 >> 24;
        int seg2 = rep << 16 >> 24;
        int mask4 = 1;
        int mask3 = !(seg4 + 1);
        int mask2 = (!(seg3 + 1)) & mask3;
        int mask1 = (!(seg2 + 1)) & mask2;
        int mask = (mask4 << 24) + (mask3 << 16) + (mask2 << 8) + mask1;
        int leftOnes = 0, tmp;

        // repeat x 8
        tmp = (rep >> 7) & mask;  leftOnes += tmp;  mask = tmp;
        tmp = (rep >> 6) & mask;  leftOnes += tmp;  mask = tmp;
        tmp = (rep >> 5) & mask;  leftOnes += tmp;  mask = tmp;
        tmp = (rep >> 4) & mask;  leftOnes += tmp;  mask = tmp;
        tmp = (rep >> 3) & mask;  leftOnes += tmp;  mask = tmp;
        tmp = (rep >> 2) & mask;  leftOnes += tmp;  mask = tmp;
        tmp = (rep >> 1) & mask;  leftOnes += tmp;  mask = tmp;
        leftOnes += rep & mask;

        // divide and add
        result = leftOnes + (leftOnes >> 8) + (leftOnes >> 16) + (leftOnes >> 24);
        result &= 0xff;
        result = 34 + ~result; // 32 + ~result + 1 + 1

        return result;
    }
    ```

### float

#### `floatScale2`

Return bit-level equivalent of expression `2*f` for floating point argument `f`

-   阶码全 0: 直接将小数段左移并加到结果上
-   阶码全 1: 按要求返回原值
-   else: 阶码加一

```C
/*   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 */
unsigned floatScale2(unsigned uf) {
    unsigned result = 0;
    unsigned EXP_MASK = 0x7f800000;
    unsigned M_MASK = 0x007fffff;
    if ((uf & EXP_MASK) == 0) {
        result += (uf & M_MASK) << 1;
        result += 0x80000000 & uf;
    } else if ((uf & EXP_MASK) == EXP_MASK) {
        result = uf;
    } else {
        result = uf + 0x00800000;
    }
    return result;
}
```

#### `floatFloat2Int`

Return bit-level equivalent of expression `(int) f` for floating point argument `f`

-   `exponent <= -1`: &ensp; $v = 1.f \times 0.5$，按 `C` 语言向 0 舍入为 0
-   `exponent == 0`: &ensp; $v = 1.f$，向 0 舍入为 1，可与下面合并
-   `0 <= exponent <= 30`: &ensp; $v = 1.f \times 2^E$，构造整数 $\mathrm{i}=1f_1f_2...f_0$
    -   `0 <= exp <= 23`: &ensp; $\mathrm{i}$ 右移 `23 - exp`
    -   `23 < exp <= 30`: &ensp; $\mathrm{i}$ 左移 `exp - 23`
-   else: &ensp; 溢出，按要求返回 `0x80000000u`

```C
/*   Legal ops: Any integer/unsigned operations incl. ||, &&. Also if, while
 *   Max ops: 30
 */
int floatFloat2Int(unsigned uf) {
    int result = 0;
    unsigned EXP_MASK = 0x7f800000;
    unsigned M_MASK = 0x007fffff;
    unsigned BIAS = 127;
    int exp = ((uf & EXP_MASK) >> 23) - BIAS;

    if (exp <= -1) {
        result = 0;
    } else if (exp < 31) {
        int m = uf & M_MASK + (1 << 23);
        if (exp <= 23)
            result = m >> (23 - exp);
        else
            result = m << (exp - 23);
    } else {
        result = 0x80000000u;
    }

    if (uf >> 31 == 1)
        result = -result;

    return result;
}
```

#### `floatPower2`

Return bit-level equivalent of the expression 2.0^x

-   `x > 128`: &ensp; 正无穷 `0x7f800000`
-   `1 - B <= x <= 128`: &ensp; 直接设置阶码
-   `1 - B - 23 <= x < 128`: &ensp; 根据 `x - (1 - B)` 的值，设置尾数的相应位
-   else: &ensp; 0

```C
/*   Legal ops: Any integer/unsigned operations incl. ||, &&. Also if, while
 *   Max ops: 30
 */
unsigned floatPower2(int x) {
    unsigned result = 0;
    int MAX_EXP = 128;
    if (x >= MAX_EXP) {
        result = 0x7f800000;
    } else if (x >= 1 - 127) {
        result += (x + 127) << 23;
    } else if (x >= 1 - 127 - 23) {
        int fBitBias = -(x + 126);
        result += 1 << 23 >> fBitBias;
    } else {
        result = 0;
    }
    return result;
}
```
