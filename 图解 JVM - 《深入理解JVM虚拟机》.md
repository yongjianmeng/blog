# 图解 JVM - 《深入理解JVM虚拟机》 #

这份笔记是我整理 《深入理解JVM虚拟机 - 第2版》 这本书时做的笔记，其在书本的原图上做了知识点的整理，希望能让原理更清晰，由于图片清晰度较高，而图片默认显示大小有限，所以放大到 150% （Chrome浏览器）效果最佳。

## Java虚拟机运行时数据区 ##

![](https://github.com/yongjianmeng/blog/blob/master/images/%E5%9B%BE%E8%A7%A3%20JVM%20-%20%E3%80%8A%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3JVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E3%80%8B-%200.png)

## 对象的访问定位 ##

![](https://github.com/yongjianmeng/blog/blob/master/images/%E5%9B%BE%E8%A7%A3%20JVM%20-%20%E3%80%8A%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3JVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E3%80%8B-%201.png)

## 可达性分析算法 ##

![](https://github.com/yongjianmeng/blog/blob/master/images/%E5%9B%BE%E8%A7%A3%20JVM%20-%20%E3%80%8A%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3JVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E3%80%8B-%202.png)

## 垃圾收集算法 ##

![](https://github.com/yongjianmeng/blog/blob/master/images/%E5%9B%BE%E8%A7%A3%20JVM%20-%20%E3%80%8A%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3JVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E3%80%8B-%203.png)

## HotSpot 虚拟机的垃圾收集器 ##

![](https://github.com/yongjianmeng/blog/blob/master/images/%E5%9B%BE%E8%A7%A3%20JVM%20-%20%E3%80%8A%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3JVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E3%80%8B-%204.png)

## 垃圾收集相关的常用参数 ##

![](https://github.com/yongjianmeng/blog/blob/master/images/%E5%9B%BE%E8%A7%A3%20JVM%20-%20%E3%80%8A%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3JVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E3%80%8B-%205.png)

> 欢迎转载，注明出处即可。