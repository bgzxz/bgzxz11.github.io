---
layout: post
title: PHP引用以及误区
city: 南京
tags: [tech]
---

###什么是引用

PHP中的引用可以理解成变量的别名。由于PHP的变量名是存储在符号表(symbol table)中的，变量内容是存储在堆中，引用就是用符号表中的不同符号(symbol)名称来访问同一存储内容，和Unix文件系统中的[hardlink][1]是同一个概念，比如：

	<?php
	$a = 1;
	$b = &$a; //$a与$b指向同一内容
	$b = 2;
	echo $b; //2
	echo $a; //2


###传递引用

引用传递很简单，就是一个「&」符号，比如：

	<?php
	function foo(&$a) {
	  $a = 2;
	}

	$b = 1;
	foo($b);
	echo $b; //2

###返回引用

大多数情况下并不需要返回引用来提高性能，zend引擎会自己进行优化，但是如果你非得返回引用得话，可以按照以下方式来返回引用：

	<?php
	class foo {
	    public $value = 42;

	    public function &getValue() { // 需要一个"&"
	        return $this->value;
	    }
	}

	$obj = new foo;
	$myValue = &$obj->getValue(); // 还需要一个"&"，$myValue是对class foo中的$value的引用
	$obj->value = 2;              // 修改对象的$value属性
	echo $myValue;                // 输出2，$myValue与class foo中的$value值相同


###与指针的区别

引用与指针很像，但是其并不是指针，看如下的代码：

	<?php
	    $a = 0;
	    $b = &a;
	    echo $a; //0
	    unset($b);
	    echo $a; //0

由于$b只是$a的别名，所以即使$b被释放了，$a没有任何影响，但是指针可不是这样的，看如下代码：

	#include <stdio.h>
	int main(int argc, char const *argv[]) {
	    int a = 0;
	    int* b = &a;
		
	    printf("%i\n", a); //0
	    free(b);
	    printf("%i\n", a); //*** error for object 0x7fff6350da08: pointer being freed was not allocated
	}

由于b是指向a的指针，所以释放了b的内存之后，再访问a就会出现错误，比较明显的说明了PHP引用与C指针的区别。

###对象与引用

在PHP中使用对象的时候，大家总是被告知“对象是按照引用传递的”，其实这是个误区。PHP的对象变量存储的是此对象的一个标示符，在传递对象的时候，其实传递的就是这个标示符，而并不是引用，看如下代码：

	<?php
	$a = new A;
	$b = $a;    
	$b->testA = 2;

	/*
	 * 此时$a,$b的关系：
	 *        +-----------+      +-----------------+
	 * $a --> | object id | ---> | object(Class A) |
	 *        +-----------+      +-----------------+
	 *                               ^
	 *        +-----------+          |
	 * $b --> | object id | ---------+
	 *        +-----------+    
	 *
	 *
	 */

	$c = new B;
	$a = $c;
	$a->testB = "Changed Class B";

	/*
	 * 此时$a,$b,$c的关系：
	 *        +-----------+      +-----------------+
	 * $b --> | object id | ---> | object(Class A) |
	 *        +-----------+      +-----------------+
	 *                               
	 *        +------------+          
	 * $a --> | object id2 | -------------+
	 *        +------------+              |
	 *                                    v
	 *        +------------+      +-----------------+
	 * $c --> | object id2 | ---> | object(Class B) |
	 *        +------------+      +-----------------+
	 */
	 
	echo "object a: "; var_dump($a); //["testB"]=> string(15) "Changed Class B"
	echo "object b: "; var_dump($b); //["testA"] => int(2)
	echo "object c: "; var_dump($c); //["testB"]=> string(15) "Changed Class B"

如果对象是按照引用传递的，那么$a, $b, $c输出的内容应该一样，事实上结果并非如此。 看下面通过引用传递对象的列子：

	<?php
	$aa = new A;
	$bb = &$aa;  // 引用 
	$bb->testA = 2;

	/*
	 * 此时$aa, $bb的关系：
	 *
	 *         +-----------+      +-----------------+
	 * $bb --> | object id | ---> | object(Class A) |
	 *         +-----------+      +-----------------+
	 *              ^                  
	 *              |
	 * $aa ---------+ 
	 *
	 *
	 */

	$cc = new B;
	$aa = $cc;
	$aa->testB = "Changed Class B";

	/*
	 * 此时$aa, $bb, $cc的关系：
	 *
	 *         +-----------+      +-----------------+
	 *         | object id | ---> | object(Class A) |
	 *         +-----------+      +-----------------+
	 *              
	 * $bb ---->-----+      
	 *               |
	 * $aa ---->-----+
	 *               |  
	 *               v   
	 *         +------------+      
	 *         | object id2 | --------------+ 
	 *         +------------+               |
	 *                                      v
	 *         +------------+      +-----------------+
	 * $cc --> | object id2 | ---> | object(Class B) |
	 *         +------------+      +-----------------+
	 */

	echo "object aa: "; var_dump($aa); //["testB"]=>string(15) "Changed Class B"
	echo "object bb: "; var_dump($bb); //["testB"]=>string(15) "Changed Class B"
	echo "object cc: "; var_dump($cc); //["testB"]=>string(15) "Changed Class B"

此时$aa，$bb，$cc三者内容完全一样，所以可以看出对象并不是按照引用传递，要尽快走出这个误区。

参考：

[http://www.php.net/manual/en/language.references.php](http://www.php.net/manual/en/language.references.php)
[http://www.php.net/manual/en/language.oop5.references.php](http://www.php.net/manual/en/language.oop5.references.php)

[1]: http://en.wikipedia.org/wiki/Hard_link "Hard link"
