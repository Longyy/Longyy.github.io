# 插入排序算法

# 插入排序算法



```go
package main

import "fmt"

//插入排序（Insertion Sort）
func InsertionSort(a []int, n int) {
	//数组个数为1，已经有序
	if n == 1 {
		return
	}
	//从第二个元素开始遍历后面每一个元素，寻找插入点
	for i := 1; i < n; i++ {
		value := a[i]
		//j标识了有序数组的末尾边界
		j := i - 1
		//从后往前遍历有序数组中的元素
		for ; j >= 0; j-- {
			//将不满足插入点位的元素后移
			if a[j] > value {
				a[j+1] = a[j]
			} else {
				//如果初始时，待插入元素大于等于有序数组中最大的元素，则插入点位已确定
				break
			}
		}
		//将待插入元素放入插入点位
		a[j+1] = value
	}
}
func main() {
	test := []int{3,5,2,6,1,8}
	InsertionSort(test, 6)
	fmt.Println(test)
}
```
#输出
`
[1 2 3 5 6 8
`
