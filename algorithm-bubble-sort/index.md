# 冒泡排序算法

#冒泡排序算法

```go
package main

import "fmt"

//冒泡排序
func BubbleSort(a []int32, n int32) {
	//循环每一轮后，有序序列长度加一
	for i := int32(0); i < n; i++ {
		//记录每一轮是否有数据交换
		flag := false
		//从头遍历数据至 n - i - 1 位置， 因为后 i 个数"已经有序"， 且因为是与后序数据比较，所以最后一个数不必遍历，-1
		for j := int32(0); j < n - i - 1; j++ {
			//默认正序排序，当前一个数 大于 后一个数时，交换位置
			if a[j] > a[j+1] {
				temp := a[j+1]
				a[j+1] = a[j]
				a[j] = temp
				flag = true
			}
		}
		//如果本轮没有数据交换，则已全部有序
		if !flag {
			break
		}
	}
	fmt.Println(a)
}
func main() {
	test := []int32{3,2,4,5,1,7,4,2, 8,0}
	BubbleSort(test, 10)
}
```


