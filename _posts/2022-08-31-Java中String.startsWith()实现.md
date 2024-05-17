---
title: 2022-08-21-Java中String.startsWith()实现.md

author: Vi_error

description: 一道极为简单的算法题引发的源码实现复习

categories:

- Java

tags:

- [Java,LeetCode]

---

# 原题目

Leetcode原题地址 : [1455. 检查单词是否为句中其他单词的前缀](https://leetcode.cn/problems/check-if-a-word-occurs-as-a-prefix-of-any-word-in-a-sentence/)

还是业务代码写多了，在看到这道题的时候，还奇怪了一下这种题目有什么好出的，然后飞快给出了ac代码。

```java
package com.vivi.algorithm.leetcode.easy.code;

/**
 * @author: Vi_error
 * @date: 2022/8/21
 */
public class No1455IsPrefixOfWord {

  public int isPrefixOfWord(String sentence, String searchWord) {
    String[] s = sentence.split(" ");
    for (int i = 0; i < s.length; i++) {
      if (s[i].startsWith(searchWord)) {
        return i + 1;
      }
    }
    return -1;
  }
}
```

然后就去看题解，发现Java语言的题解没有使用startsWith()，还是自己写了比对代码()。

```java
class Solution {

  public int isPrefixOfWord(String sentence, String searchWord) {
    int n = sentence.length(), index = 1, start = 0, end = 0;
    while (start < n) {
      while (end < n && sentence.charAt(end) != ' ') {
        end++;
      }
      if (isPrefix(sentence, start, end, searchWord)) {
        return index;
      }

      index++;
      end++;
      start = end;
    }
    return -1;
  }

  public boolean isPrefix(String sentence, int start, int end, String searchWord) {
    for (int i = 0; i < searchWord.length(); i++) {
      if (start + i >= end || sentence.charAt(start + i) != searchWord.charAt(i)) {
        return false;
      }
    }
    return true;
  }
}
```

官方题解中没有使用任何包装好的算法，将split和startsWith都自行实现了。看到这里的时候我开始回忆这两个方法的实现，竟然也要像做题一样重新去想。

# startsWith()

调用入口如下：

```
public boolean startsWith(String prefix) {
    return startsWith(prefix, 0);
}
```

实现方法如下：
```
public boolean startsWith(String prefix, int toffset) {
    char ta[] = value;
    int to = toffset;
    char pa[] = prefix.value;
    int po = 0;
    int pc = prefix.value.length;
    // Note: toffset might be near -1>>>1.
    if ((toffset < 0) || (toffset > value.length - pc)) {
        return false;
    }
    while (--pc >= 0) {
        if (ta[to++] != pa[po++]) {
            return false;
        }
    }
    return true;
}
```

转char遍历实现比较，O(n)

# split()

看完startsWith()之后又顺带看了一下split底层实现，基于正则的实现，防御性做得很不错。

```
 public String[] split(String regex, int limit) {
        /* fastpath if the regex is a
         (1)one-char String and this character is not one of the
            RegEx's meta characters ".$|()[{^?*+\\", or
         (2)two-char String and the first char is the backslash and
            the second is not the ascii digit or ascii letter.
         */
        char ch = 0;
        if (((regex.value.length == 1 &&
             ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
             (regex.length() == 2 &&
              regex.charAt(0) == '\\' &&
              (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
              ((ch-'a')|('z'-ch)) < 0 &&
              ((ch-'A')|('Z'-ch)) < 0)) &&
            (ch < Character.MIN_HIGH_SURROGATE ||
             ch > Character.MAX_LOW_SURROGATE))
        {
            int off = 0;
            int next = 0;
            boolean limited = limit > 0;
            ArrayList<String> list = new ArrayList<>();
            while ((next = indexOf(ch, off)) != -1) {
                if (!limited || list.size() < limit - 1) {
                    list.add(substring(off, next));
                    off = next + 1;
                } else {    // last one
                    //assert (list.size() == limit - 1);
                    list.add(substring(off, value.length));
                    off = value.length;
                    break;
                }
            }
            // If no match was found, return this
            if (off == 0)
                return new String[]{this};

            // Add remaining segment
            if (!limited || list.size() < limit)
                list.add(substring(off, value.length));

            // Construct result
            int resultSize = list.size();
            if (limit == 0) {
                while (resultSize > 0 && list.get(resultSize - 1).isEmpty()) {
                    resultSize--;
                }
            }
            String[] result = new String[resultSize];
            return list.subList(0, resultSize).toArray(result);
        }
        return Pattern.compile(regex).split(this, limit);
    }
```