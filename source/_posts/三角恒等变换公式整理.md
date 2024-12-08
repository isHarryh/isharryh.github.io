---
title: 三角恒等变换公式整理
toc: true
categories:
  - Maths
tags: [笔记]
date: 2024-11-28 19:12:00
updated: 2024-11-28 19:12:00
---

以下是常见的三角恒等变换公式，整理出来以便参考：

<!-- more -->

## 基本公式

### 与“1”相关的恒等式

$$
\sin^2 x + \cos^2 x = 1
$$

> **口诀：**  
> Sin 方加 Cos 方等于 1。

$$
1 + \tan^2 x = \sec^2 x
$$
$$
1 + \cot^2 x = \csc^2 x
$$

> **口诀：**  
> 1 加切方等于割方。

### 弦割互换的倒数关系

$$
\csc x = \frac{1}{\sin x}, \quad \sec x = \frac{1}{\cos x}
$$

> **口诀：**  
> 余割正弦分之一，正割余弦分之一。

## 和角与差角

$$
\sin(x \pm y) = \sin x \cos y \pm \cos x \sin y
$$
$$
\cos(x \pm y) = \cos x \cos y \mp \sin x \sin y
$$
$$
\tan(x \pm y) = \frac{\tan x \pm \tan y}{1 \mp \tan x \tan y} \quad (\tan x \cdot \tan y \neq 1)
$$

## 倍角与半角

### 二倍角公式

$$
\sin(2x) = 2\sin x \cos x
$$
$$
\cos(2x) = \cos^2 x - \sin^2 x = 2\cos^2 x - 1 = 1 - 2\sin^2 x
$$
$$
\tan(2x) = \frac{2\tan x}{1 - \tan^2 x} \quad (\tan x \neq \pm 1)
$$

### 三倍角公式

$$
\sin(3x) = 3\sin x - 4\sin^3 x
$$
$$
\cos(3x) = 4\cos^3 x - 3\cos x
$$
$$
\tan(3x) = \frac{3\tan x - \tan^3 x}{1 - 3\tan^2 x} \quad (\tan x \neq \pm \frac{\sqrt{3}}{3})
$$

### 半角公式

$$
\sin^2 x = \frac{1 - \cos(2x)}{2}
$$
$$
\cos^2 x = \frac{1 + \cos(2x)}{2}
$$
$$
\tan^2 x = \frac{1 - \cos(2x)}{1 + \cos(2x)} \quad (\cos(2x) \neq -1)
$$

## 和差化积与积化和差

### 和差化积公式

$$
\sin x + \sin y = 2\sin\left(\frac{x + y}{2}\right)\cos\left(\frac{x - y}{2}\right)
$$
$$
\cos x + \cos y = 2\cos\left(\frac{x + y}{2}\right)\cos\left(\frac{x - y}{2}\right)
$$
$$
\sin x - \sin y = 2\cos\left(\frac{x + y}{2}\right)\sin\left(\frac{x - y}{2}\right)
$$
$$
\cos x - \cos y = -2\sin\left(\frac{x + y}{2}\right)\sin\left(\frac{x - y}{2}\right)
$$

> **口诀：**  
> Sin 加 Sin 等于 2 Sin Co，Co 加 Co 等于 2 Co Co。  
> Sin 减 Sin 等于 2 Co Sin，Co 减 Co 等于 -2 Sin Sin。

### 积化和差公式

$$
\sin x \cos y = \frac{1}{2} [\sin(x + y) + \sin(x - y)]
$$
$$
\cos x \cos y = \frac{1}{2} [\cos(x + y) + \cos(x - y)]
$$
$$
\sin x \sin y = -\frac{1}{2} [\cos(x + y) - \cos(x - y)]
$$
