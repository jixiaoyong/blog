---
title: Java中三种常用的排序方法
abbrlink: 1fd30f6e
date: 2016-12-20 23:25:05
---



今天重新学习类三种排序方法，按照排序速度依次是冒泡排序，选择排序和插入排序。
以下示例皆为从小到大的排序

# 1.冒泡排序
每一次比较都可能要交换元素。
冒泡排序的思想是：
每一轮开始的时候，将第一个元素（a）开始与其后的元素（b）依次进行比较，将较大的元素（设为m）放到后面，并将m与其后的另外一个元素继续进行比较，直到最后一个没有排好序的元素。
在接下来一轮的排序中，刚才以及之前选出来的、已经排好顺序的最大值不用参与排序。
依次类推，总共遍历n-1轮，即可完成排序。
具体代码如下：


     void bubble(int[] arr){
    	int temp;
    	for (int i = 0; i < arr.length - 1; i++) {
    		for (int j = 0; j < arr.length - i - 1; j++) {
    			if (arr[j] > arr[j + 1]) {
    				temp = arr[j];
    				arr[j] = arr[j + 1];
    				arr[j + 1] = temp;
    			}
    		}
    	}
    	
    	System.out.println("\n--bubble :");
    	for (int i = 0; i < arr.length; i++) {
    		System.out.print(arr[i] + " ");
    	}
    }


# 2.选择排序
每次比较的时候不交换
选择排序的思想：
每次比较的时候找到的两个数中的较大值并记下其位置，等到当前一轮的遍历完成之后，将最后一个未排序元素与这一轮遍历找到的最大值交换
最多交换n-1次
代码如下：

       void select(int[] arr){
    
    	for (int i = 0; i < arr.length; i++) {
    		int maxIndex = 0;
    		int temp = 0;
    	
    		for (int j = 1; j < arr.length - i; j++) {
    			if (arr[maxIndex] < arr[j]) {
    				maxIndex = j;
    			}
    		}
    		
    		temp = arr[maxIndex];
    		arr[maxIndex] = arr[arr.length - i - 1];
    		arr[arr.length - i - 1] = temp;
    	}


		System.out.println("\n--select :");
	
		for (int i = 0; i < arr.length; i++) {
			System.out.print(arr[i] + " ");
		}
	}

# 3.插入排序法
插入排序法思想：
将待排序的元素分为有序和无序两种，刚开始排序的时候假设只有第一个元素是有序的，其余n-1个元素都是无序的；
排序开始的时，将无序部分的一个元素（a）与有序部分的最后一个元素（b）进行比较，如果a<b，则将a与b交换，再将a与下一个有序元素进行比较；否则，将a加到b后面，作为有序部分的最后一个元素。
接着再从无序部分取出一个元素与有序部分的元素依次比较，直达所有元素都为有序元素。
遍历n-1次
代码如下：

```java
    void insertSort(int[] arr){

	for (int i = 1; i < arr.length; i++) {
		int instertValue = arr[i];
		
		for (int j = i - 1; j >= 0; j--) {
			if (instertValue < arr[j]) {
				arr[j+1] = arr[j];
				arr[j] = instertValue;
			}else {
				break;
			}
		}
	}
	
	/* 第二种表示形式
	for (int i = 1; i < arr.length; i++) {
		int instertVal = arr[i];
		int index = i - 1;
		
		while (index >= 0 && instertVal < arr[index]) {
			arr[index + 1] = arr[index];
			index--;
		}
		arr[index + 1] = instertVal;
	}		
	*/

	System.out.println("\n--insertSort :");
	for (int i = 0; i < arr.length; i++) {
		System.out.print(arr[i] + " ");
	}
}
```

