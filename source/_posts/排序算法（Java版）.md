title: 排序算法（Java版）
date: 2015-05-27 22:31:20
categories: 算法
tags:
- 算法
---

在《算法分析与设计》(Algorithm Design: Foundations, Analysis, and Internet Examples)中，作者将各种排序算法分成了三类，分别为PQ-Sort简单排序模式、基于分治设计思想以及基于散列法，觉得相当的有趣，于是就按照这种思路将所学的各种算法整理了一下。

### 一、PQ-Sort简单排序模式
所谓PQ-Sort简单排序模式，其思路大概就是基于这样一种思路:将一组无序的元素插入到优先队列，然后通过一系列的removeMin操作达到使序列从无序到有序的过程。在算法实现中优先队列往往不是真正的存在，它大多数时候只是作为一个抽象的概念。这中算法实现起来简单，但是效率一般不高，通常都要O(n^2)。
以下算法都是PQ-Sort模式的排序算法

#### 1、选择排序
最简单的排序。令A[1…n]是一个元素的数组，首先找到A中最小的元素，将其放到A[1]中，然后在剩下的n-1个元素中找到最小的元素，将其放到A[2]中，重复此过程直至找到第二大元素将其放在A[n-1]中。
Java代码如下:
{% code %}
public class SelectionSort{
	public static int[] selectionSort(int[] array){
		for(int i=0; i < array.length-1; i++){
			int currentMinIndex = i;
			for(int j=i+1; j < array.length; j++){
				if(array[j] < array[currentMinIndex]){
					currentMinIndex = j;
				}
			}
			if(currentMinIndex != i){
				int tmpValue = array[i];
				array[i] = array[currentMinIndex];
				array[currentMinIndex] = tmpValue;
			}
		}
		return array;
	}
}
{% endcode %}
无论数组的初始化排序如何，执行算法比较的次数为n(n-1)/2。
<!--more-->

#### 2、插入排序
插入排序的思路决定了它是一种受序列是否有序有较大影响的一种排序算法。其思路是，从大小为1的数组A[1]开始，接下来将A[2]插入到A[1]前面或者后面，继续这一过程直至序列有序。
Java代码如下：
{% code %}
public class InsertionSort{
	public static int[] insertionSort(int[] array){
		for(int i=1; i < array.length; i++){
			int currentMin = array[i];
			int sortedIndex = i - 1;
			while(sortedIndex >= 0 && array[sortedIndex] > currentMin){
				array[sortedIndex+1] = array[sortedIndex];
				sortedIndex--;
			}
			array[sortedIndex+1] = currentMin;
		}
		return array;
	}
}
{% endcode %}
比较次数为n(n-1)/2。

####3、堆排序
堆排序在PQ-Sort模式算法中是效率比较高的算法，算法只需要O(nlogn)的时间已经O(1) 的空间。在SelectionSort中对n个元素的排序过程中，算法的效率消耗在n-1次循环中算法每次都必须在剩下的元素进行线性搜索来找出最小值（最大值）。堆排序对此进行了改进。堆排序的过程如下，首先将A变换成堆，在A中，每个元素的键值为其本身，即key(A[i])=A[i]。在初始堆中A[1]是最大的元素，将A[1]与A[n]交换，此时A[n]是整个序列的最大元素，在利用堆的siftDown过程将A[1…n-1]转化成堆。重复这个过程直至堆的大小变成1，此时A[1]是整个序列中最小的元素。
Java代码如下：
{% code %}
public class HeapSort{
	public static int[] heapSort(int array){
		Heap heap = new Heap(array);
		heap.makeHeap();
		for(int i=array.length-1; i > 0; i--){
			heap.deleteMax();
		}
		int  sortedArray = heap.getArray();
		return sortedArray;
	}
}

class Heap{
	private int[] array;
	private int lastIndex;

	public Heap(){
		this.array = null;
	}

	public Heap(int[] array){
		this.array = array;
		this.lastIndex = array.length-1;
	}

	public void siftUp(){
		if(lastIndex == 0)
			return;
		for(int i=lastIndex; lastIndex > 0; i = i/2){
			if(array[i] > array[i/2]){
				int tmpValue = array[i];
				array[i] = array[i/2];
				array[i/2] = tmpValue;
			}else{
				break;
			}
		}
	}

	public void siftDown(int firstIndex){
		if(2*firstIndex > lastIndex)
				return;
		for(int i=2*firstIndex; i <= lastIndex; i=i*2){
			if(i+1 <= lastIndex && array[i+1] > array[i]){
				i=i+1;
			}
			if(array[i/2] < array[i]){
				int tmpValue = array[i];
				array[i] = array[i/2];
				array[i/2] = tmpValue;
			}else{
				break;
			}
		}
	}

	public int delete(int index){
		int deleteValue = array[index];
		int lastValue = array[lastIndex];
		if(index == lastIndex)
			return lastValue;

		lastIndex = lastIndex-1;
		array[index] = lastValue;

		if(deleteValue > lastValue){
			siftDown(index);
		}else{
			siftUp();
		}
		return deleteValue;
	}

	public int deleteMax(){
		int max = array[0];
		this.delete(0);
		this.array[this.lastIndex+1] = max;
		return max;
	}

	public int[] makeHeap(){
		for(int i=array.length/2; i >= 0; i--){
			this.siftDown(i);
		}
		return array;
	}

	public int[] getArray(){
		return this.array;
	}
}
{% endcode %}
排序时间开销为O(nlogn)。



### 二、基于分治的排序模式
分治是一种强有力的算法设计技术，基本思路是把一个问题实例分成若干子实例（大多数是两个），并分别递归地解决每个子实例，然后把这些子实例的解组合起来，得到原来问题实例的解。

#### 1、合并排序
合并排序的过程是先将输入的序列分成两个，接着分别对这两个子序列排序，然后只是将它们合并起来得到所希望的排序数组。在算法执行过程中，对子数组的排序算法是可以任意选择的，通常递归使用合并排序本身。
比如对于数组A=[9, 4, 5, 2, 1, 7, 4, 6, 8]，则分成的两个子数组为A1=[9, 4, 5, 2]以及A2=[1, 7, 4, 6, 8]。
合并排序有两种形式，其中一种是非递归的合并排序，这种在某些算法书中称为BottomUpSort(自底向上排序），另外一种是递归排序，自顶向下，容易理解，以下是自顶向上递归合并排序的代码：
{% code %}
public class MergeSort{
	private int[] array;

	public MergeSort(){
		this.array = null;
	}

	public MergeSort(int[] array){
		this.array = array;
	}

	public void mergeSort(int low, int height){
		if( low < height){
			int mid = (low+height)/2;
			mergeSort(low, mid);
			mergeSort(mid+1, height);
			merge(low, mid, height);
		}
	}

	private void merge(int low, int mid, int height){
		int[] tmpArray = new int[height-low+1];
		int lowIndex = low;
		int heightIndex = mid+1;

		int i=0;
		while(lowIndex <= mid && heightIndex <= height){
			if(array[lowIndex] > array[heightIndex]){
				tmpArray[i] = array[heightIndex];
				heightIndex++;
				i++;
			}else if(array[lowIndex] < array[heightIndex]){
				tmpArray[i] = array[lowIndex];
				lowIndex++;
				i++;
			}else{
				tmpArray[i] = array[lowIndex];
				lowIndex++;
				i++;
				tmpArray[i] = array[heightIndex];
				heightIndex++;
				i++;
			}
		}
		while(lowIndex <= mid){
			tmpArray[i] = array[lowIndex];
			lowIndex++;
			i++;
		}
		while(heightIndex <= height){
			tmpArray[i] = array[heightIndex];
			heightIndex++;
			i++;
		}

		i=0;
		lowIndex = low;
		while(i < tmpArray.length && lowIndex <= height){
			array[lowIndex] = tmpArray[i];
			lowIndex++;
			i++;
		}

		tmpArray = null;
	}

	public int[] getSortArray(){
		return this.array;
	}
}
{% endcode %}
算法运行时间为Θ(nlogn)，但需要Θ(n)的空间。

###2、快速排序
快速排序是一个非常流行且高效的排序算法，和归并排序一样具有O(nlogn)的平均运行时间，并且可以在原位上排序，无需辅助空间。
快速排序的基础是划分算法。设A[low…high]是一个包含了n个数的数组，且x=A[low]，利用划分算法，使得小于或等于x的元素在x前面，所有大于x的元素在x后面。例如，A=[5, 3, 9, 2, 7, 1, 8]，low=0， high=6，经过排序之后A=[1, 3, 2, 5, 7, 9, 8]，此时w=3。这即为围绕主元x的拆分算法。
因为经过主元拆分之后x所在的位置恰好是排序结果的正确位置，所知只需对其前后的子数组经行递归调用划分算法，即可使数组有序。
算法实现如下:
{% code %}
public class QuickSort{
	private int[] array;

	public QuickSort(){
		this.array = null;
	}

	public QuickSort(int[] array){
		this.array = array;
	}

	public void quickSort(int low, int high){
		if( low < high){
			int pivot = split(low, high);
			quickSort(low, pivot-1);
			quickSort(pivot+1, high);
		}
	}

	public int[] getSortedArray(){
		return this.array;
	}

	private int split(int low, int high){
		int lowIndex = low;
		int lowValue = array[low];
		for(int i=low+1; i <= high; i++){
			if(array[i] < lowValue){
				lowIndex++;
				if(lowIndex != i){
					int tmpValue = array[i];
					array[i] = array[lowIndex];
					array[lowIndex] = tmpValue;
				}
			}
		}

		int tmpValue = array[lowIndex];
		array[lowIndex] = array[low];
		array[low] = tmpValue;
		return lowIndex;
	}
}
{% endcode %}


###三、基于散列法的桶组法模式
基于散列法的排序只能针对特定类型的元素序列的排序。通过对元素的格式做限制性的假设，避免使用比较。
####1、基数排序
假如A[1…n]是一个有n个数的表，每个数恰好都有k位数，就是说A数组中每个数都具有d(k)d(k-1)…d(1)的形式，其中d(i)是0～9中的一个数字，那么存在一种算法不是对这些元素本身经行排序，而是按照每个整数上第k位上整数的大小来排序。基数排序就是按照这种思路来来对序列进行排序的。在实现基数排序的过程中，必须根据元素的最低位来分配到合适的奇数表中，然后将这些表再合并起来，从低位到高位排序，最终可以完成基数排序。
这种排序方式对于位数不大的整型序列进行排序十分有效。
Java代码实现如下：
{% code %}
import java.util.List;
import java.util.ArrayList;
public class RadixSort{
	public static int[] radixSort(int[] array){
		int max = maxElement(array);
		int lastIndex = bitLength(max);

		ArrayList<Integer> arrayList = new ArrayList<>();
		for(int n: array){
			arrayList.add(n);
		}

		List<ArrayList<Integer>> buckets = new ArrayList<ArrayList<Integer>>();
		for(int i=0; i < 10; i++){
			ArrayList<Integer> bucket = new ArrayList<>();
			buckets.add(bucket);
		}

		for(int i=0; i < lastIndex; i++){
			ArrayList<Integer> bucket = null;
			while( !arrayList.isEmpty()){
				int element = arrayList.get(0);
				arrayList.remove(0);
				int bit = nthBit(element, i);
				bucket = buckets.get(bit);
				bucket.add(element);
				buckets.set(bit, bucket);
			}

			for(int j=0; j < 10; j++){
				bucket = buckets.get(j);
				for(int n: bucket){
					arrayList.add(n);
				}
				bucket.clear();
				buckets.set(j, bucket);
			}
		}

		for(int i=0; i < arrayList.size(); i++){
			array[i] = arrayList.get(i);
		}

		return array;
	}

	private static int maxElement(int[] array){
		int max = array[0];
		for(int i=1; i < array.length; i++){
			if(array[i] > max){
				max = array[i];
			}
		}
		return max;
	}

	private static int bitLength(int num){
		int length = 0;
		while( num > 0){
			length ++;
			num /= 10;
		}
		return length;
	}

	private static int nthBit(int num, int nth){
		int digit = 0;
		for(int i=0; i <= nth; i++){
			digit  = num % 10;
			num /= num;
		}
		return digit;
	}
}
{% endcode %}
运行时间为O(nlogn)，但需要O(kn)空间。
