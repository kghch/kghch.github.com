---
layout: post
title: valid number
category: algorithm
---

解法一：

1. skip the leading whitespaces;
2. then, skip the sign;
3. for, until 不等于’.’ or is not a digit,必须保证’.’数量<=1, digit数量 >= 1;
4. skip the trailing whitespaces;