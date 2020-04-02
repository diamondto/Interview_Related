# 字节跳动效率工程一面  

- css position 定位  
  position:   
  1. absolute：第一个非 static 定位的祖先元素，如果没有就相对视窗  
  2. relative：相对自身  
  3. fixed:  相对于视窗  
  4. static:  默认布局，不加 position 默认就是 static  
  5. sticky:  粘性，比如滑出一定的距离之后固定(fixed)  

- 类数组转换真正的数组  
  1. Array.from()  
  2. 展开运算符 + 解构  
  3. 借用数组上的方法  
    Array.prototype.slice.call(xxx)  
    Array.prototype.splice.call(xxx, 0)  

- 值判断  
  if ([])  
  if ({})  
  基本数据类型 - 变量和变量对应的值都存储在栈中  
  复杂数据类型 - 变量存储在栈内存，值存储在堆内存中  
  不严格相等 == 和严格相等 === 转换规则 -> MDN  
  MDN 转换规则 👉 https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Equality_comparisons_and_sameness  
  