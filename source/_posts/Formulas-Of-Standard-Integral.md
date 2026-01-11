---
title: 标准积分公式整理
toc: true
categories:
  - Maths
tags: [数学]
date: 2024-12-09 21:52:00
updated: 2024-12-09 21:52:00
---

以下是常用的积分公式，用于处理复杂的有理函数、三角函数、指数函数和对数函数积分问题，整理出来以便参考：

<!-- more -->

## 基本函数积分

以下积分公式是基于“原函数”的性质直接推得的。

### 积分 幂函数

- 一般幂：（$n \neq -1$）
  $$ \int x^n \ \text d x = \frac{x^{n+1}}{n+1} + C$$
  $$ \int (-x)^n \ \text d x = \frac{(-1)^n x^{n+1}}{n+1} + C $$
- 倒数幂：（$n = -1$）
  $$ \int \frac{1}{x} \ \text d x = \ln |x| + C $$
- 常函数：（$n = 0$）
  $$ \int a \ \text d x = ax + C $$

### 积分 指数函数

- 一般指数：
  $$ \int a^x \ \text d x = \frac{a^x}{\ln a} + C \quad (a > 0, a \neq 1) $$
- 自然指数：
  $$ \int e^x \ \text d x = e^x + C $$

### 积分 对数函数

- 自然对数：
  $$ \int \ln x \ \text d x = x \ln x - x + C $$

### 积分 三角函数

$$
\begin{align*}
\int \sin x \ \text d x &= -\cos x + C \cr
\int \cos x \ \text d x &= \sin x + C \cr
\int \tan x \ \text d x &= -\ln |\cos x| + C \cr
&= \ln |\sec x| + C
\end{align*}
$$

### 积分到 三角函数

$$
\begin{align*}
\int \sec^2 x \ \text d x &= \tan x + C \cr
\int \csc^2 x \ \text d x &= -\cot x + C \cr
\int \sec x \tan x \ \text d x &= \sec x + C \cr
\int \csc x \cot x \ \text d x &= -\csc x + C
\end{align*}
$$

### 积分到 反三角函数

$$
\begin{align*}
\int \frac{1}{\sqrt{1 - x^2}} \ \text d x &= \arcsin x + C \cr
\int \frac{-1}{\sqrt{1 - x^2}} \ \text d x &= \arccos x + C \cr
\int \frac{1}{1 + x^2} \ \text d x &= \arctan x + C \cr
\int \frac{-1}{1 + x^2} \ \text d x &= \text{arccot} \ x + C \cr
\int \frac{1}{x \sqrt{x^2 - 1}} \ \text d x &= \text{arcsec} \ x + C \cr
\int \frac{-1}{x \sqrt{x^2 - 1}} \ \text d x &= \text{arccsc} \ x + C
\end{align*}
$$

## 有理函数积分公式

$$
\begin{align*}
\int \frac{1}{a^2 + x^2} \ \text d x &= \frac{1}{a} \arctan\left(\frac{x}{a}\right) + C \cr
\int \frac{1}{\sqrt{a^2 - x^2}} \ \text d x &= \arcsin\left(\frac{x}{a}\right) + C \cr
\int \frac{1}{\sqrt{1+x^2}} \ \text d x &= \ln (x+\sqrt{1+x^2}) + C \cr
\int \frac{1}{x^2 - a^2} \ \text d x &= \frac{1}{2a} \ln\left|\frac{x - a}{x + a}\right| + C \cr
\int \frac{1}{a^2 - x^2} \ \text d x &= \frac{1}{2a} \ln\left|\frac{a + x}{a - x}\right| + C \cr
\int \frac{x}{x^2 + a^2} \ \text d x &= \frac{1}{2} \ln(x^2 + a^2) + C \cr
\int \frac{1}{(x^2 + a^2)^n} \ \text d x \quad (n \geq 2) & ：通过递推公式计算 \cr
\int \frac{x}{\sqrt{x^2 + a^2}} \ \text d x &= \sqrt{x^2 + a^2} + C
\end{align*}
$$

## 混合型积分公式

### 自然指数×三角函数

$$
\int e^{ax} \sin bx \ \text d x = \frac{e^{ax}}{a^2 + b^2} (a \sin bx - b \cos bx) + C
$$
$$
\int e^{ax} \cos bx \ \text d x = \frac{e^{ax}}{a^2 + b^2} (a \cos bx + b \sin bx) + C
$$

### 幂函数×对数函数

$$
\int x^n \ln x \ \text d x = \frac{x^{n+1}}{n+1} (\ln x - \frac{1}{n+1}) + C \quad (n \neq -1)
$$
