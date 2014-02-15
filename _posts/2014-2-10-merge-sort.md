---
layout: post
title: 归并排序
category: algorithm
excerpt: 基础排序算法中，归并排序是很值得学习的。不仅在于其分治思想的应用，也在于其优化空间复杂度的研究。
tags: [算法]
---
{% include JB/setup %}

## 归并排序实现 ##
归并排序是基础的排序算法之一，算法课中最先接触的一类算法。对于任意的有N个元素的数组，将其排序成功的时间复杂度是NlogN，空间复杂度是N。传统的两路归并是需要额外的辅助空间的，尤其是N很大时，这种开销有时候会变的不可接受。而当我们尝试实现原地排序（不需要或者利用很小的辅助空间）时，算法实现的复杂度也随之上升了。在<a href="#tips">经典</a>中，Knuth提到了一种很巧妙的方法，通过在排序数组中构造出辅助空间。  
假设，有2个已排序数组A和B，按照sqrt(N)的大小，分成若干段。这时A和B的最后一段(A(k)、B(l))就包含了2个数组中可能的最大元素。此时如果做一个merge（归并）动作，按照从小到大的顺序，将它们排列到A(k)和B(l)中，B(l)因为存放较大值，可明确一点是，它现在拥有了2个数组的最大元素（归并后集合的最大值），将它用来做辅助空间有一个优势，就是不管怎么merge，它所包含的元素是不会变化的。
下面的归并动作，就是传统的2路归并了。这里需要点预处理，将（除了B(l)）数据段都排好序（或者说，按照数据段中的最小元素整体排序）。  

	为什么呢？这里其实就涉及到算法的合理性。来张图，就可以一目了然。如何论证，A，B大元素、小元素的最终排布
<img src="{{ ASSET_PATH }}/images/merge_sort_1.jpg"/>
在完成两两相邻块的归并后，除了B(l)，其他数据已经是OK了，所以我们还需要对B(l)进行一次调整。按照这种思路，是可以实现原地归并排序的，但在实现过程中，还是有点复杂的。  
除此之外有一个相对容易理解以及实现的方法，就是在归并过程中，移动一个“区间”，而非原来的单个元素。不多解释了，直接将代码贴出来。  

	private static void merge(Comparable[] array, int start, int mid, int end) {
		// 定义两个移动的指针
		int i = start;
		int j = mid + 1;
		int old_j = -1;
		while (i <= mid && j <= end) {
			if (SortHelper.less(array[i], array[j])) {
				if (old_j != -1) {
					reverse(array, i, old_j, j - 1);
					i = i + j - old_j;
					mid = mid + j - old_j;
					old_j = -1;
				}
				i++;
			} else {
				if (old_j == -1)
					old_j = j;
				else 
					j++;
			}
		}
		if (old_j != -1)
			reverse(array, i, old_j, j - 1);
	}

	private static void reverse(Comparable[] array, int i, int old_j, int j) {
		exch(array, i, old_j - 1);
		exch(array, old_j, j);
		exch(array, i, j);
	}

	private static void exch(Comparable[] array, int i, int j) {
		int mid = (j - i) / 2 + i;
		while (i <= mid) {
			SortHelper.exch(array, i++, j--);
		}
	} 
在这段代码中，有一个明显的问题是效率下降，主要集中在了交换区间的过程中。这里用的“手摇法”，由于存在3次旋转，在极端情况下，时间复杂度很高。当然，如何交换区间，也可以设计出精巧的策略，需要仔细研究。  
**PS:** 将长度为N的数组排序，任何基于比较的算法至少需要lg(N!)~NlgN次比较次数。这其实就是在研究归并排序过程中，得出的一个重要结论，它指示了我们，不管怎么调整算法，基于比较的排序有一个比较次数的下限。

----------
<a name="tips" href="http://www-cs-faculty.stanford.edu/~uno/taocp.html">经典：这里指的是 The Art of Computer Programming（TAOCP）。</a>