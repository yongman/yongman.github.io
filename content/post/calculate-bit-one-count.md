---
title: 计算二进制中比特位1的个数
categories: [算法]
date: 2018-08-17 09:36:39
---

比如一个32位的无符号整型，如果要统计其中二进制中1的个数，直接从最低位遍历，不管二进制中1的个数有多少，时间复杂度是一样的。

还有比较快速的计算方式：

```
int countOneBit(unsigned int num) {
  int count;
  for (count=0;num>0;count++) {
    num &= num - 1;
  }
  return count;
}
```

每次位与运算都会将其中最后一位比特1置0，直到所有位都是0结束。

对应优化在[Tidis](https://github.com/yongman/tidis)中bitcount命令，commit [6f1f7e9](https://github.com/yongman/tidis/commit/6f1f7e920d3987decd2f1afc238c0fa54399de4f)
