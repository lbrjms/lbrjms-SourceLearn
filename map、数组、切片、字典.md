# js weakmap weakset
https://www.cnblogs.com/xiaonian8/p/13700201.html
可以看到weakmap的会随着键的释放而释放 map中就不会  vue判断属性是否被监听过 用的就是weak
```
function usageSize() {
  const used = process.memoryUsage().heapUsed;
  return Math.round((used / 1024 / 1024) * 100) / 100 + "M";
}

global.gc();
console.log(usageSize()); // ≈ 3.19M

let arr = new Array(10 * 1024 * 1024);
const map = new WeakMap();

map.set(arr, 1);
global.gc();
console.log(usageSize()); // ≈ 83.2M

arr = null;
global.gc();
console.log(usageSize()); // ≈ 3.2M
```

# go 数组、切片
切片就是可变数组 
* copy函数在拷贝数据时永远以小容量为准
* append函数每次给切片扩容都会按照原有切片容量*2的方式扩容
```
	arr := []int{1, 2, 3}
	arr[1] = 23
	fmt.Println(arr)
	mArr1 := arr[2:]
	mArr := make([]int, 3)
	mArr = append(mArr, 12)
	fmt.Println(mArr, mArr1)
	//copy(mArr1, mArr)
	copy(mArr, mArr1)
	fmt.Println(mArr, mArr1)
```
