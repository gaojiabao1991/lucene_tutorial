# 索引

## 索引构建

让我们用下面的文本构建一个倒排索引，注意以下几点

* 分词
* 停用词
* 词项归一化

### 文本

	doc0: "Lucene is a high-performance text search engine lucene is easy to use"
	doc1: "Apache Lucene is an open source project available for free download"
	doc2: "Lucene is a full-featured text search engine"
	doc3: "solr is a free full-text search engine written in Java"
	
### 简单版索引

	apache	[1]
	available	[1]
	download	[1]
	engine	[0, 2, 3]
	free	[1, 3]
	full-featured	[2]
	full-text	[3]
	high-performance	[0]
	java	[3]
	lucene	[0, 1, 2]
	open	[1]
	project	[1]
	search	[0, 2, 3]
	solr	[3]
	source	[1]
	text	[0, 2]
	written	[3]
	
### 完整版索引

	apache	1	{1=1[0]}
	available	1	{1=1[7]}
	download	1	{1=1[10]}
	easy	1	{0=1[9]}
	engine	3	{0=1[6], 2=1[6], 3=1[6]}
	free	2	{1=1[9], 3=1[3]}
	full-featured	1	{2=1[3]}
	full-text	1	{3=1[4]}
	high-performance	1	{0=1[3]}
	java	1	{3=1[9]}
	lucene	3	{0=2[0, 7], 1=1[1], 2=1[0]}
	open	1	{1=1[4]}
	project	1	{1=1[6]}
	search	3	{0=1[5], 2=1[5], 3=1[5]}
	solr	1	{3=1[0]}
	source	1	{1=1[5]}
	text	2	{0=1[4], 2=1[4]}
	to	1	{0=1[10]}
	use	1	{0=1[11]}
	written	1	{3=1[7]}
	
### 索引的存储

#### 倒排表：SkipList

![](http://www.sheeva.cn/files/2017-12-26_2052.png)

> Skip lists are a probabilistic data structure that seem likely to supplant balanced trees as the implementation method of choice for many applications. Skip list algorithms have the same asymptotic expected time bounds as balanced trees and are simpler, faster and use less space.

> 跳跃列表是在很多应用中有可能替代平衡树而作为实现方法的一种数据结构。跳跃列表的算法有同平衡树一样的渐进的预期时间边界，并且更简单、更快速和使用更少的空间。

—— 引用发明者的话

##### 倒排表压缩

	1. delta-encode
	2. split into block of N=128 values
	3. bit packing per block
	4. if remaining docs, encode with vInt

----

	1. 先进行delta差值压缩
	2. 压缩后把值按顺序分组，目前是每个128个值分为一组
	3. 每组值按位压缩(参考类PackedInts)
	4. 如果有剩余不够一组的, 用vInt压缩

假设 N=4 来举例:

	原始倒排表： 1, 3, 4, 6, 8, 20, 22, 26, 30, 31 
	
	Delta差值压缩：1, 2, 1, 2, 2, 12, 2, 4, 4, 1
	
	按N值分组：[1, 2, 1, 2] [2, 12, 2, 4] 4, 1
	
压缩每一组值(PackedInts)	

	org.apache.lucene.util.packed.PackedInts.Encoder  （Interface）
	
	//Read iterations * valueCount() values from values, encode them and write iterations * blockCount() blocks into blocks.
	
	void encode(int[] values,
            int valuesOffset,
            long[] blocks,
            int blocksOffset,
            int iterations)
			
			
![](http://www.sheeva.cn/files/2017-12-27_1709.png)	
-----

    BulkOperation implements Encoder
	
	new BulkOperationPacked1(),
    new BulkOperationPacked2(),
    new BulkOperationPacked3(),
	
	...
	
    new BulkOperationPacked23(),
    new BulkOperationPacked24(),
    new BulkOperationPacked(25),
    new BulkOperationPacked(26),
	
	...
	
    new BulkOperationPacked(60),
    new BulkOperationPacked(61),
    new BulkOperationPacked(62),
    new BulkOperationPacked(63),
    new BulkOperationPacked(64),
	
-----

	public BulkOperationPacked(int bitsPerValue) {
        ...
	}
	    


vInt Encoding Example：

|  Value |    Byte1 |    Byte2 |    Byte3 |
|--------+----------+----------+----------|
|      0 | 00000000 |          |          |
|      1 | 00000001 |          |          |
|      2 | 00000010 |          |          |
|    ... |          |          |          |
|    127 | 01111111 |          |          |
|    128 | 10000000 | 00000001 |          |
|    129 | 10000001 | 00000001 |          |
|    130 | 10000010 | 00000001 |          |
|    ... |          |          |          |
| 16,383 | 11111111 | 01111111 |          |
| 16,384 | 10000000 | 10000000 | 00000001 |
| 16,385 | 10000001 | 10000000 | 00000001 |


> Question: vInt与utf-8的对比

| Unicode符号范围(十六进制) | UTF-8编码(二进制)                   |
|---------------------------+-------------------------------------|
| 0000 0000-0000 007F       | 0xxxxxxx                            |
| 0000 0080-0000 07FF       | 110xxxxx 10xxxxxx                   |
| 0000 0800-0000 FFFF       | 1110xxxx 10xxxxxx 10xxxxxx          |
| 0001 0000-0010 FFFF       | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx |



#### 词典：FST

*注：N=词典长度；n=查询字符串长度*

| 数据结构          | 优缺点                                                                    |
|-------------------+---------------------------------------------------------------------------|
| 有序Array         | 使用二分查找，log(N)复杂度；不方便更新                                    |
| TreeMap           | log(N)复杂度；内存消耗大，几乎是原始数据的三倍                            |
| SkipList          | 接近log(N)复杂度；内存消耗小于TreeMap但是大于原始数据                     |
| Trie              | n 复杂度,使用公共前缀减少内存消耗；对于存储大量没有公共前缀的字符串将非常消耗内存 |
| Double Array Trie | n 复杂度，通过重叠不同节点的children_node充分利用空节点从而节省内存； |
| Finite State Transducers(FST) | |

FST:

1. 结构紧凑，通过对词典中前缀和后缀的重复利用，压缩了存储空间
2. O(n)的查询时间复杂度

![](http://www.sheeva.cn/files/2017-12-27_1152.png)


##### 数字类型的值

Lucene 对数值类型的值在索引时有特殊的处理

假设我要使用倒排索引的结构保存数字，如何高效实现常见的范围查询？

例如：

> 1. 在招聘场景下，要找出工作经验3~5年的求职者
> 2. 在电商场景下，要找出价格在5000~10000的笔记本电脑

假设某个字段需要索引以下值：

421,423,445,446,448,521,522,632,633,634,641,642,644

使用常规的字典生成的倒排索引：

	421 -> [1]
	423 -> [2]
	445 -> [3]
	446 -> [3]
	448 -> [4]
	521 -> [5]
	522 -> [7]
	632 -> [5]
	633 -> [6]
	634 -> [7]
	641 -> [5]
	642 -> [6]
	644 -> [7]
	
此时要搜索range[423,642]，需要依次获取423,445,446,...,634,641,642的倒排表并合并

-----

> Better Solution


![](http://www.sheeva.cn/files/2017-12-27_1719.png)

索引的时候，除了索引每个数值之外，还把商也索引起来,每个商的倒排表是所有叶子节点所包含的文档

	42 -> [1, 2]
	44 -> [3, 4]
	52 -> [5, 7]
	63 -> [5, 6]
	64 -> [5, 6 , 7]
	4 -> [1, 2, 3, 4]
	5 -> [5, 7]
	6 -> [5, 6, 7]
	
现在当我们搜索range[423,642]时，只需要取423、44、5、63、641、642 这几个节点的倒排表就行了

#### I/O操作

**org.apache.lucene.store.Directory**

> Java's i/o APIs not used directly, but rather all i/o is through this API. This permits things such as:

> * implementation of RAM-based indices;

> * implementation indices stored in a database, via JDBC;

> * implementation of an index as a single file;

**子类结构**

	Directory
        > BaseDirectory
		    > RAMDirectory
	        > FSDirectory
			    > SimpleFSDirectory
				> NIOFSDirectory
				> MMapDirectory


## 索引删除：Bitmap 

**Example:**

delete a doc： [1,1,1,1,1,1] -> [1,0,1,1,1,1]    

*1 = live, 0 = deleted*


	* Deletion = turn a bit off
	* Ignore deleted documents when searching and merging
	* Merge policies favor segments with many deletions

	* 删除 = 改变一个bit
	* 在搜索和分段合并时忽略被删除的文档
	* 分段合并策略倾向于合并被删除文档多的segment



## 索引分段

> 为什么需要索引分段

### 索引合并

> 为什么需要索引合并

    two basic algorithms:
        1. make an index for a single document
        2. merge a set of indices

    incremental algorithm:
        * maintain a stack of segment indices
        * create index for each incoming document
        * push new indexes onto the stack
        * let b=10 be the merge factor; M=∞

        for (size = 1; size < M; size *= b) {
          if (there are b indexes with size docs on top of the stack) {
           pop them off the stack;
           merge them into a single index;
           push the merged index onto the stack;
         } else {
           break;
         }
        } 

    optimization: single-doc indexes kept in RAM, saves system calls

**Example:**


* b = 3
* 11 documents indexed
* stack has four indexes
* grayed indexes have been deleted
* 5 merges have occurred

![](http://www.sheeva.cn/files/2017-12-27_1757.png)
