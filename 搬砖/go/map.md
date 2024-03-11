1.  delete(m, "key"), 可以在 for 循环中安全删除键值，不能新增
2.  循环无序
3.  循环遍历的值是临时变量，修改是无意义的，同 `slice` 一样，该临时变量每次是一个值，所以循环取这个临时变脸的地址，所有指针最终会指向同一个值
4. 如果 `map` 的值是 `struct`  `map[key].val = val` 是无意义的，正确方式  
	a. 整个值全部赋值 `map[key] = struct{val:''}`
	b. 值保存指针 `map[key]*strcut` ,然后 `map[key].val = val`