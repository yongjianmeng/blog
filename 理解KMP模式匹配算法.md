# 理解KMP模式匹配算法 #

网上已经有很多关于 KMP 模式匹配算法的讲解，但是大都晦涩难懂，因为有的一些讲解是直接上代码，还有的一些里面包含了很多公式推理。在把 KMP 模式匹配算法理解透彻后，我决定写一篇教程把如何理解 KMP 模式匹配算法尽量通俗易懂地向大家讲解，希望对大家有帮助。

> 这篇文章适合看过 KMP 模式匹配算法但不能理解透彻的同学。

## 朴素模式匹配算法 ##

在正式进入主题之前，我们先用朴素模式匹配算法做个引子。

### 例子 ###

假设我们要从主串 t = "goodgoole" 中找到子串 p = "google" 的位置，通常我们最先想到的应该是以一种暴力的方法解决，也就是常说的朴素模式匹配算法，步骤如下：

- 第一步：从 t 的第 0 位开始匹配，发现 t 和 p 前面的三位 "goo" 是相等的，但到了第 3 位时，"d" 与 "g" 不相等了，所以第 0 位匹配失败。如下图所示，**红叉表示不相等，绿条表示相等**。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-1.png)

<p align="center">图-1</p>

- 第二步：移动 p，尝试从 t 的第 1 位开始匹配，发现 t 的第 1 位是 "o"，与 p 的第 0 位 "g" 不相等，匹配失败。如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-2.png)

<p align="center">图-2</p>

- 第三步：继续移动 p，尝试从 t 的第 2 位开始匹配，和第二步一样，匹配失败。如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-3.png)

<p align="center">图-3</p>

- 第四步：继续移动 p，尝试从 t 的第 3 位开始匹配，和第三步一样，匹配失败。如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-4.png)

<p align="center">图-4</p>

- 第五步：继续移动 p，尝试从 t 的第 4 位开始匹配，这时发现所有的字符都是相等的，匹配成功。如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-5.png)

<p align="center">图-5</p>

### 时间复杂度分析 ###

设主串 t 的长度为 n，子串 p 长度为 m。朴素模式匹配算法的**最好情况**是第一步就匹配成功了，这时的时间复杂度为 *O(m)*，即只要遍历一遍子串 p 即可。**稍差一些**，每次匹配都如第二、三、四步一样，都是首字母不匹配，那么就不必遍历子串 p 了，例如在 "abcdefgoogle" 中查找 "google"，这时时间复杂度为 O(n + m)，即只要遍历一遍主串 t 和子串 p 即可。根据等概率原则，平均是 （n + m）/ 2 次查找，因此平均时间复杂度为 *O(n + m)*。**最坏的情况**是每次不相等的匹配都发生在子串 p 的最后一个字符，例如主串 t = "00000000000000000000000000000000001"，子串 p = "0000001"，在最后成功匹配之前，每次失败的匹配都要遍历到 p 的最后一位时才发现不匹配，即存在很多无效的重复的匹配，这时的时间复杂度为 *O((n - m + 1) * m)*。

## KMP模式匹配算法 ##

KMP算法的诞生正是为了避免这种无效的重复的匹配。KMP模式匹配算法由三个人名组成，即 D.E.Knuth、J.H.Morris 和 V.R.Pratt，并没有其它的意思。那么KMP算法是如何避免这种无效的重复的匹配呢？

### 例子 ###

假设主串 t = "abcdefgab..."，子串 p = "abcdex"，**朴素模式匹配算法**在第 0 位的匹配如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-6.png)

<p align="center">图-6</p>

第 0  ~ 4 位都是相等的，但到第 5 位时 "x" 与 "f" 不相等，按照朴素模式匹配算法就会移动 p，把它的首字母 p[0] （即 "a"） 继续与 t[1]、t[2]、t[3]、t[4]（即 "b"，"c"，"d"，"e"）作比较，发现它们都与 p[0]不相等。但是，既然我们知道在子串 p 中的首字母 "a" 不与其后面的 "bcdex" 中任意一字符相等，那么这些比较就重复了，我们完全可以跳过它们，直接跳到比较 p[0] 与 t[5]，如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-7.png)

<p align="center">图-7</p>

注意，虽然我们知道在子串 p 中 p[0] 与 p[5] 不相等，在第一步比较中我们知道了 p[5] 与 t[5] 不相等，但我们并不能推导出 p[0] 不等于 t[5]，因此我们仍然要比较 p[0] 与 t[5]，比较之后才知道它们相不相等。

上面的例子中运气非常的好，子串 p 中的所有字符都不相等，我们完全可以跳过已经相等的部分。但如果碰到在子串 p 中有重复字符的时候该怎么办呢？那么这个时候我们**就不能一次性跳过这么多个字符了**。

如下图所示，假设主串 t = "abcabaabcab..."，子串 p = "abcabb"，子串 p 中有重复的字符。还是和之前的例子一样，两个字符串在最后一个字符比较时不相等。那么这个时候我们能直接跳过相等的部分吗？肯定不行的，**因为有相同的字符出现在子串中，我们不能保证跳过之后会错过正确答案，因此我们不能全部跳过!!!**。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-8.png)

<p align="center">图-8</p>

再仔细看一下上图高亮的地方，我们发现，p[3]p[4] 与 t[3]t[4] 是相等的（都是"ab"），而 p[1]p[2] 也与 t[3]t[4] 相等，这时我们虽然不能完全跳过相等的部分（即 p[0] 跳到直接与 t[5] 作比较），但**退而求其次**，我们可以**移动子串 p **至 p[0]p[1] 与 t[3]t[4] 重合，这时我们不需要比较  p[0]p[1] 与 t[3]t[4]，因为我们确定它们是相等的，而且它们之前的部分也不需要比较，因为我们知道，p[0] （即"a" ）与 t[1] （即"b"）、t[2]（即"c"）是不相等的，而是直接开始比较 p[2] 与 t[5]。这样就直接跳过了 p[0] 与 t[1]、 p[0] 与 t[2] 、p[0] 与 t[3]、 p[1] 与 [t4] 的比较。如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-9.png)

<p align="center">图-9</p>

这也是网上很多教程没有说明为什么要求出 next 数组（暂时不用知道什么是next数组，后面我们会说明）的原因：通俗地说，**因为有了next数组，我们就可以知道，如果某一位不相等了，我们应该怎么移动子串 p， 使得我们能避免重复比较**。

那么什么是next数组呢？

next 数组由子串 p 通过计算得出，通过它我们能够知道，当主串 t 与 子串 p 的某一位不匹配时，我们应该移动子串 p 中的第几位来与 主串 t 继续进行比较。**以此来跳过重复比较部分**（有的教程会用移动子串 p 上的指针 j 来说明，我们通过移动子串也是同样的道理，而且更能说明其中的原理）。

例如上面的例子中，子串 p = "abcabb" 的 next 数组在没有改进（我们后面会说明改进的方法）之前是 [-1, 0, 0, 0, 1, 2]。

那么这个 next 数组有什么意义呢？其实除了第一个 -1 之外，其他的几位意义是相同的。
例如上面的例子就是利用 next 数组中的最后一位 2 来移动子串 p。当比较到子串 p 的最后一位与 t 的某一位不相等时，从 next 数组中我们得到 子串 p 最后一位对应的数值是 next[5] （即 2），那么我们应该移动子串 p，让子串 p 的第 2 位（即 p[2]）与刚才不相等的地方（即 t[5]）继续进行比较。

再假设，当比较到如下图所示的位置不相同时（即 p[4] 与 t[4] 不相等）：

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-10.png)

<p align="center">图-10</p>

我们去 next 数组中查找，next[4] 为1，这时我们移动子串 p，让子串 p 的第 1 位（即 p[1]）与刚才不相等的地方（即 t[4]）继续进行比较，移动结果如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-11.png)

<p align="center">图-11</p>

继续假设，当比较到如下图所示的位置不相同时（即 p[3] 与 t[3] 不相等）：

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-12.png)

<p align="center">图-12</p>

我们去 next 数组中查找，next[3] 为1，这时我们移动子串 p，让子串 p 的第 0 位（即 p[0]）与刚才不相等的地方（即 t[3]）继续进行比较，移动结果如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-13.png)

<p align="center">图-13</p>

next[1]、next[2] 同理，那么next[0]有什么意义呢？在比较时，如果在 p[0] 位置就不相等了，在 p[0] 之前没有字符可以用来继续比较了，如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-14.png)

<p align="center">图-14</p>

那咋办呢？很好解决，把子串 p 移动一位来继续进行比较，因为第 0 位就不相等了，我们只能移动 p 继续进行比较。如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-15.png)

<p align="center">图-15</p>

知道了 next 数组的作用，那么我们怎么通过子串 p 来计算出它的 next 数组呢？

求解 next 数组的公式如下图所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-16.png)

<p align="center">图-16</p>

**第一种情况**很好理解，在 next 数组中，当 索引 j 为 0 时，对应的 next[0] 恒等于 -1。所以所有的 next 数组的第一个元素都为 -1。

**第二种情况**相对难一些，它的意思是，当我们求 index[j] 时，如果能在子串第 j 个字符之前（即不包括第 j 个字符）的字符串中找到前缀字符串与后缀字符串相等的情况，那么取出其中最长的一种情况，它对应的 k 值就是 index[j] 的值，可能这么说有点难懂，那么请看下图。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-17.png)

<p align="center">图-17</p>

在上图中，我们要求 index[j] 的值，那么在子串 p 第 j 个字符之前的字符串，即 "abcab" 中，我们能找出前缀字符串与后缀字符串相等的情况 "abcab"，这种情况也是最长的一种，所以取其 k 值，即 2 作为 index[j] 的值。如果子串 p = "abccccab"，那么 index[7] 之前的字符串为 "abcccca"，在这个字符串中，前缀字符串和后缀字符串相等的情况是 "abcccca"，此时 k 值为1，因此 index[7] 为 1。如果子串 p = "aaaaaaab"，那么 index[7] 之前的字符串为 "aaaaaaa"，在这个字符串中，前缀字符串和后缀字符串都是 "aaaaaaa"，此时 k 值为6，因此 index[7] 为 6。

**第三种情况**是当上面两种情况都不满足时考虑的，这时设置 index 值为 0。聪明的你应该能想到这种情况吧？没错，它出现的情况就是在没有相等的字符时。例如 p = "abcdefgh"，那么 index[7] 之前的字符串为 "abcdefg"，在这个字符串中，没有前缀字符串和后缀字符串相等的情况，此时 k 值为0，因此 index[7] 为 0。

## 代码实现 - Java版本 ##
求 index 数组的代码如下所示。有了上面对 index 数组的解释，理解这段代码应该很容易了。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-18.png)

那么在朴素模式匹配算法中怎么利用 index 数组呢？KPM 模式匹配算法代码如下所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-19.png)

在代码高亮部分是重点。当在匹配时，如果两个字符不相等，那么我们就去 index 数组中查找，当在子串 p 中第 j 位不相等时，应该如何调整 j 的值（即我们上文中所提到的移动子串 p，道理是一样的，可以理解为**运动是相对的**），调整 j 值后从新的位置开始继续进行比较直到主串 t 或子串 p 到尾了。
我们前面说过可以改进 KMP 模式匹配算法。那么为什么要改进它呢？

如上图10到图11，从图10到图11的移动是完全没有意义的，因为在图10中，p[4]（即"b"）与 t[4] （即"c"）不相等，而我们根据 index 数组得知要移动过来的 p[1] 和 p[4] 是相等的，那么 p[1] 肯定与t[4] 不相等，移动过来作比较肯定是得到不相等的结果，我们就还得继续从 index 数组中查询一次。为了防止这种情况的发生，我们在生成 index 数组的时候可以进行一个判断，当两个字符相等时就继续跳过一次（即再查找一次 index 数组）。改进后的代码如下所示。

![](https://github.com/yongjianmeng/blog/blob/master/images/%E7%90%86%E8%A7%A3KMP%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D%E7%AE%97%E6%B3%95-20.png)

## 总结 ##
总的来说KMP算法还是很好理解的，难点主要在于index数组如何计算得出，通过这篇文章希望对大家理解KMP算法有帮助，有问题欢迎提交 PR 讨论。

> 欢迎转载，注明出处即可。