#crash 
越界异常(NSRangeException)也是比较常出现的异常，有如下几种类型:

1. 数组最大下标处理错误
	比如数组长度count, index的下标范围[0, count -1], 在开发时，可能index的最大值超过数组的范围；
2. 下标的值是其他变量赋值
	这样会有很大的不确定性， 可能是一个很大的整数值
3. 使用空数组
	如果一个数组刚刚初始化，还是空的，就对它进行相关操作

所以，为了避免NSRangeException的发生，必须对传入的index参数进行合法性检查，是否在集合数据的个数范围内。