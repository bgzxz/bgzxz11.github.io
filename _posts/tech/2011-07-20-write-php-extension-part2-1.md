---
layout: post
title: PHP扩展编写第二步：参数，数组，以及ZVAL
city: 南京
tags: [translate]
---

###介绍

在这个系列教程的第一部分，你已经了解了一个PHP扩展的基本框架结构。你声明了一个简单的函数，这个函数向调用它的脚本返回静态和动态的值，定义了INI配置项，以及声明了内部的值（全局变量）。在这个教程中，你将会知道如何接收传递到你函数中的参数，并且认识到**PHP**和**Zend Engine**在内部是如何管理变量的。

###接收参数

不像在用户空间的代码那样，一个内部函数的参数实际上不会声明在函数的头部。相反，参数列表的引用会传递到每个函数中 – 不管参数传递了没有 – 接下来函数就可以让Zend Engine把这些参数变成可以使用的变量。

让我们来看一下这个过程，我们定义了一个新函数，`hello_greetme()`，这个函数会接收1个参数，然后这个参数会和一些其他的问候语一起输出。像之前一样，我将在三个地方添加代码：

在**php_hello.h**文件靠近其他函数原型的地方：

	PHP_FUNCTION(hello_greetme);

在hello.c文件中hello_functions结构体的结尾处：

	PHP_FE(hello_bool, NULL)
	    PHP_FE(hello_null, NULL)
	    PHP_FE(hello_greetme, NULL)
	    {NULL, NULL, NULL}
	};

在靠近hello.c文件结尾处，其他函数的后面：

	PHP_FUNCTION(hello_greetme)
	{
	    char *name;
	    int name_len;

	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &name, &name_len) == FAILURE) {
	        RETURN_NULL();
	    }

	    php_printf("Hello %s ", name);
	    RETURN_TRUE;
	}


`zend_parse_parameters()`函数大部分的代码看起来几乎都一样。`ZEND_NUM_ARGS()`向Zend Engine提供关于接收到的参数数量，`TSRMLS_CC`被用来保证线程安全，最后函数会返回SUCCESS或者FAILURE。在普通情况下`zend_parse_parameters()`将会返回`SUCCESS`；如果一个调用脚本传递的参数数量超过了函数定义的参数数量，或者传递的参数不能转换成合适的数据类型，Zend将会自动的输出一个错误信息，然后函数会优雅地把控制权返回给调用脚本。

在这个例子你使用“s”来指明这个函数只能接收一个参数，并且这个参数可以被转换成`string`数据类型，存入通过引用传递的`char*`变量中。

注意一个`int`变量也以引用的方式传递给了`zend_parse_parameters()`。这个允许`Zend Engine`来提供这个字符串的长度有多少字节，所以二进制安全的函数就不需要依赖`strlen(name)`来检测字符串的长度。事实上，使用`strlen(name)`有可能得不到正确的结果，因为`name`可能在字符串末尾前面包含一个或者多个`NULL`字符。

一旦你的函数接收到了`name`这个参数，函数接下来要做的事情就是输出一句正式问候语，并把`name`参数的值作为其中的一部分。注意到我们使用了`php_printf()`而没有使用更加熟悉的`printf()`函数。使用这个函数非常重要，有几个原因。第一，它允许通过PHP的输出缓冲（output buffering）机制对输出的字符串进行处理，这个机制实际上除了对数据进行缓冲之外，它还会执行一些额外的处理比如gzip压缩。第二，当以CLI或者CGI方式使用PHP的时候，`stdout`是一个数据输出的完美目的地，但是大多数SAPI希望通过一个指定的管道或者套接字来进行输出。因此，如果想简单地使用`printf()`来把数据输出到`stdout`可能会导致数据丢失，数据发送顺序混乱，或者数据错误，因为它的bypassed 预处理（bypassed preprocessing）。

最后这个函数通过简单地返回TRUE来把控制权交给那个调用程序。当然你也可以不用明确的返回一个值（默认为NULL），但是这么做是非常糟糕的。如果一个函数没有任何有用的信息需要报告的话，那么就返回`TRUE`，简单的说一句：“一切OK，我完成了你让我完成的任务”。

因为PHP字符串有可能事实上包含`NULL`，所以要想输出一个二进制安全的字符串，其中包含`NULL`，并且还有`NULL`后边的字符，那么方法是把`php_printf()`语句替换成如下代码块：

	php_printf("Hello ");
	PHPWRITE(name, name_len);
	php_printf(" ");

这个代码块使用了`php_printf()`来处理不包含`NULL`字符的字符串，但是用了另一个宏 –`PHPWRITE`来处理用户提供的字符串。这个宏接受由`zend_parse_parameters()`提供的长度参数（`name_len`），所以`name`参数的整体内容可以不用担心NULL而放心得输出。

`zend_parse_parameters()`也可以处理可选参数。在下一个例子中，你将会创建一个函数，这个函数接收一个long(PHP的整型)，一个double (浮点型)，和一个可选的Boolean值。在用户空间声明这个函数的话可能看起来如下：

	function hello_add($a, $b, $return_long = false) {
	    $sum = (int)$a + (float)$b;

	    if ($return_long) {
	        return intval($sum);
	    } else {
	        return floatval($sum);
	    }
	}

在C语言中，这个函数看起来如下（当你想把它加到hello.c中的时候，不要忘记在php_hello.h和hello_functions[]中加入这个函数实体声明）：

	PHP_FUNCTION(hello_add)
	{
	    long a;
	    double b;
	    zend_bool return_long = 0;

	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "ld|b", &a, &b, &return_long) == FAILURE) {
	        RETURN_NULL();
	    }

	    if (return_long) {
	        RETURN_LONG(a + b);
	    } else {
	        RETURN_DOUBLE(a + b);
	    }
	}

这一次，你的数据类型字符串这样读：“我需要一个(`l`)ong类型的参数，然后一个(`d`)ouble类型的”。下一个管道字符说明接下来的参数列表是可选的。如果一个可选的参数在函数调用过程中没有被传递，那么`zend_parse_parameters()`不会改变已经传给它的参数值。最后那个`b`显然是Boolean类型。在数据类型字符串之后，`a`，`b`，以及`return_long`以引用的方式传递进来，所以`zend_parse_parameters()`可以用参数值来填充它们。

警告：尽管`int`和`long`在32位平台上是可以相互替换着使用的，但是如果这么做的话，当你的代码在64位硬件上重新编译的时候就会非常危险。所以记住用`long`来处理long类型，用`int`来处理字符串长度。

Table 1 给出了各种数据类型，以及可以在`zend_parse_parameters()`中使用，与这些类型相对应的字母和C语言数据类型：

	Table 1: Types and letter codes used in zend_parse_parameters()
	Type	Code	Variable Type
	Boolean	 b       zend_bool
	Long	 l       long
	Double	 d       double
	String	 s       char*, int
	Resource r       zval*
	Array	a        zval*
	Object	o        zval*
	zval	z        zval*


你可能注意到了<em>Table 1</em>中最后四个类型都返回了相同的数据类型 – 一个`zval*`。一个`zval`，就像你将要看到的那样，是用来存储PHP中所有用户空间变量的真实数据类型。三个“复杂”的数据类型，Resource, Array和Object，当他们的数据类型字母标示在`zend_parse_parameters()`中被使用的时候，Zend Engine会对其进行类型检查，但是它们在C语言中没有相对应的数据类型，所以不会有任何转换会被实际执行。

###ZVAL

`zval`和普通的PHP用户空间变量，将会是最难理解，绞尽你脑汁的概念。它们也将会是最重要的概念。开始之前，让我们看看一个`zval`的结构：

	struct {
	    union {
	        long lval;
	        double dval;
	        struct {
	            char *val;
	            int len;
	        } str;

	        HashTable *ht;
	        zend_object_value obj;

	    } value;

	    zend_uint refcount;
	    zend_uchar type;
	    zend_uchar is_ref;

	} zval;

就像你看见的，每个`zval`通常有三个基本的元素：`type`，`is_ref`，`refcount`。`is_ref`和`refcount`将会在这个教程的后面介绍；现在先让我们关注一下`type`。

到现在为止你应该已经熟悉了PHP的8个数据类型。它们中的7个在Table 1中列出来了，再加上`NULL`，尽管事实上它在字面上表示什么都没有。对于一个特定的`zval`来说，它的类型是可以通过以下三个便利的宏中的一个来进行检查：`Z_TYPE(zval)`，`Z_TYPE_P(zval*)`，或者`Z_TYPE_PP(zval**)`。这个三个唯一的不同就是对传递变量的引用的层级要求不同。对于`_P` 和 `_PP`这种命名的惯例在其他宏中也会出现，比如你将要看见的`*VAL`宏。

`type`决定了`zval``value`联合体的中那一部分被使用。下面的代码片段演示了一个简单版本的`var_dump()`：

	PHP_FUNCTION(hello_dump)
	{
	    zval *uservar;
	
	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", uservar) == FAILURE) {

	        RETURN_NULL();

	    }

	    switch (Z_TYPE_P(uservar)) {

	        case IS_NULL:
	            php_printf("NULL ");
	            break;

	        case IS_BOOL:
	            php_printf("Boolean: %s ", Z_LVAL_P(uservar) ? "TRUE" : "FALSE");
	            break;

	        case IS_LONG:
	            php_printf("Long: %ld ", Z_LVAL_P(uservar));
	            break;

	        case IS_DOUBLE:
	            php_printf("Double: %f ", Z_DVAL_P(uservar));
	            break;

	        case IS_STRING:
	            php_printf("String: ");
	            PHPWRITE(Z_STRVAL_P(uservar), Z_STRLEN_P(uservar));
	            php_printf(" ");
	            break;

	        case IS_RESOURCE:
	            php_printf("Resource ");
	            break;

	        case IS_ARRAY:
	            php_printf("Array ");
	            break;

	        case IS_OBJECT:
	            php_printf("Object ");
	            break;

	        default:
	            php_printf("Unknown ");

	    }

	    RETURN_TRUE;
	}

就像你所看到的，`Boolean`数据类型使用了和`long`数据类型一样的内部宏。就像你在这个系列教程的第一部分中使用的`RETURN_BOOL()`那样，`FALSE`用0来代表，`TRUE`用1来代表。

当你使用`zend_parse_parameters()`来要求一个指定了数据类型的参数的时候，比如`string`，`Zend Engine`将会去检查输入变量的数据类型。如果类型匹配，Zend会简单地把参数值简单的传递到`zval`中对应的部分。如果不匹配，那么Zend会使用type-juggling规则对输入变量进行类型转换。

对你之前实现的函数`hello_greetme()`进行修改，把其分成几个小片段：

	PHP_FUNCTION(hello_greetme)
	{
	    zval *zname;

	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", &zname) == FAILURE) {
	        RETURN_NULL();
	    }

	    convert_to_string(zname);
	    php_printf("Hello ");
	    PHPWRITE(Z_STRVAL_P(zname), Z_STRLEN_P(zname));
	    php_printf(" ");

	    RETURN_TRUE;
	}

这一次，`zend_parse_parameters()`简单的接收一个不定数据类型的PHP变量（`zval`），然后这个函数明确的把这个变量转换成字符串类型(类似于`$zname = (string)$zname; `)，然后使用Z_STRVAL_P宏得到`zname`的值，并输出。就像你可能猜到的，和其他数据类型`bool`，`long`和`double`相对应的`convert_to_*()`函数也是存在的。

###创建ZVAL

到现在为止，你所使用的zval都是由Zend Engine来分配和释放的。但是无论如何，有时候，自己创建zval是很必要的。看一下如下的代码块：

	{
	    zval *temp;
	
	    ALLOC_INIT_ZVAL(temp);

	    Z_TYPE_P(temp) = IS_LONG;
	    Z_LVAL_P(temp) = 1234;

	    zval_ptr_dtor(&temp);
	}

ALLOC_INIT_ZVAL()，就像它名字所表明的那样，为一个`zval*`分配内存，然后初始化为一个新的变量。当这个过程一旦完成，`Z_*_P()`宏就可以被用来设置变量的数据类型和值。`zval_ptr_dtor()`被用来做一脏活：清理分配给变量的内存。

这两个`Z_*_P()`宏的调用可以被减少到一个单独的语句：<br/><br/>`ZVAL_LONG(temp, 1234);`

对于其他数据类型，类似的宏也是存在的，并且它们有相同的语法格式，就像你在这个系列教程第一部分所见的`RETURN_*()`宏。事实上`RETURN_*()`宏就是对`RETVAL_*()`宏进行了简单的包装，`ZVAL_*()`宏也是一样的道理。下面的五个版本是等同的：

	RETURN_LONG(42);

	RETVAL_LONG(42);
	return;

	ZVAL_LONG(return_value, 42);
	return;

	Z_TYPE_P(return_value) = IS_LONG;
	Z_LVAL_P(return_value) = 42;
	return;

	return_value->type = IS_LONG;
	return_value->value.lval = 42;
	return;

如果你足够机警，那么你会考虑这些宏是怎么定义的，就像它们在`hello_long()`函数中被使用的那样。“`return_value`是从哪来的，为什么它没有通过 ALLOC_INIT_ZVAL()来进行分配？”，你可能想知道。

在你一天又一天的扩展编写过程中`return_value`被隐藏了起来，其实它是在每个`PHP_FUNCTION()`原型中定义的一个函数参数。Zend Engine会为它分配内存，然后初始化为`NULL`，所以即使你的函数没有实际的去设置这个变量，其也有了一个可以被调用程序所用的值。当你的内部函数执行完之后，Zend Engine会把这个变量的值传递给调用程序，或者如果调用程序被告知忽略这个变量，则释放掉它。

###数组

既然你在过去使用过PHP，那么你已经认识到一个数组变量的作用就是包含其他各种变量。它内部是通过一个大家都熟悉的数据结构`HashTable`来实现的。当要创建数组，并且把这些数组返回给PHP，最简单的方法就是使用Table 2中所列函数中的一个。

	Table 2: zval array creation functions

	PHP Syntax  C Syntax (arr is a zval*)   Meaning
	$arr = array(); array_init(arr);             Initialize a new array
	$arr[] = NULL;  add_next_index_null(arr);   Add a value of a given type to a numerically indexed array
	$arr[] = 42;    add_next_index_long(arr, 42);
	$arr[] = true;  add_next_index_bool(arr, 1);
	$arr[] = 3.14;  add_next_index_double(arr, 3.14);
	$arr[] = 'foo'; add_next_index_string(arr, "foo", 1);
	$arr[] = $myvar; add_next_index_zval(arr, myvar);
	$arr[0] = NULL; add_index_null(arr, 0);     Add a value of a given type to a specific index in an array
	$arr[1] = 42;   add_index_long(arr, 1, 42);
	$arr[2] = true; add_index_bool(arr, 2, 1);
	$arr[3] = 3.14; add_index_double(arr, 3, 3.14);
	$arr[4] = 'foo';add_index_string(arr, 4, "foo", 1);
	$arr[5] = $myvar; add_index_zval(arr, 5, myvar);
	$arr['abc'] = NULL; add_assoc_null(arr, "abc"); Add a value of a given type to an associatively indexed array
	$arr['def'] = 711; add_assoc_long(arr, "def", 711);
	$arr['ghi'] = true; add_assoc_bool(arr, "ghi", 1);
	$arr['jkl'] = 1.44; add_assoc_double(arr, "jkl", 1.44);
	$arr['mno'] = 'baz'; add_assoc_string(arr, "mno", "baz", 1);
	$arr['pqr'] = $myvar; add_assoc_zval(arr, "pqr", myvar);


像`RETURN_STRING()`宏一样，`add_*_string()`函数也会在最后一个参数中用0或者1来指明这个字符串内容是否需要复制。这些`add_*_string()`函数每个还有`add_*_stringl()`格式的变体。`l`表示字符串的长度将会明确地提供（不需要Zend Engine调用非二进制安全的`strval()`函数来检测）。

使用这个二进制安全格式的函数只需要简单的在那个复制参数前面指定长度即可，像这样：

	add_assoc_stringl(arr, "someStringVar", "baz", 3, 1);

在使用`add_assoc_*()`函数的时候，所有数组的key都假设不包含任何的`NULL` – `add_assoc_*()`函数本身对于key是非二进制安全。在它们之中使用包含`NULL`的key是不被鼓励的（即使这个技术已经在protected对象属性和private对象属性中使用了），但是如果必须这么做的话，当我们稍后接触`zend_hash_*()`函数的时候，你将会了解到如何更加充分地用好它。

把你刚才已经学到的东西展示一下，创建如下的一个函数，其返回一个数组的值给调用程序。确定在**php_hello.h**和`hello_functions[]`中加入适当的函数实体来声明这个函数。

	PHP_FUNCTION(hello_array)
	{
	    char *mystr;
	    zval *mysubarray;
	    array_init(return_value);

	    add_index_long(return_value, 42, 123);
	    add_next_index_string(return_value, "I should now be found at index 43", 1);
	    add_next_index_stringl(return_value, "I'm at 44!", 10, 1);

	    mystr = estrdup("Forty Five");
	    add_next_index_string(return_value, mystr, 0);
	
		add_assoc_double(return_value, "pi", 3.1415926535);

	    ALLOC_INIT_ZVAL(mysubarray);
	    array_init(mysubarray);
	    add_next_index_string(mysubarray, "hello", 1);
	    add_assoc_zval(return_value, "subarray", mysubarray);
	}

构建这个扩展，然后给出`var_dump(hello_array())`的结果：

	array(6) {
	  [42]=>
	  int(123)

	  [43]=>
	  string(33) "I should now be found at index 43"

	  [44]=>
	  string(10) "I'm at 44!"

	  [45]=>
	  string(10) "Forty Five"

	  ["pi"]=>
	  float(3.1415926535)

	  ["subarray"]=>
	  array(1) {
	    [0]=>
	    string(5) "hello"
	  }
	}

读取数组中的值意味着使用`ZENDAPI`中的`zend_hash`一类的函数把`HashTable`中的内容抽出然后转换成`zval**`。让我们以一个接收数组参数的简单函数开始：

	function hello_array_strings($arr) {
	    if (!is_array($arr)) return NULL;
	    printf("The array passed contains %d elements ", count($arr));

	    foreach($arr as $data) {
	        if (is_string($data)) echo "$data ";
	    }
	}

或者，用C语言：

	PHP_FUNCTION(hello_array_strings)
	{
	    zval *arr, **data;
	    HashTable *arr_hash;
	    HashPosition pointer;
	    int array_count;

	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "a", &arr) == FAILURE) {
	        RETURN_NULL();
	    }

	    arr_hash = Z_ARRVAL_P(arr);
	    array_count = zend_hash_num_elements(arr_hash);
	    php_printf("The array passed contains %d elements ", array_count);

	    for(zend_hash_internal_pointer_reset_ex(arr_hash, &pointer); zend_hash_get_current_data_ex(arr_hash, (void**) &data, &pointer) == SUCCESS; zend_hash_move_forward_ex(arr_hash, &pointer)) {
	        if (Z_TYPE_PP(data) == IS_STRING) {

	            PHPWRITE(Z_STRVAL_PP(data), Z_STRLEN_PP(data));

	            php_printf(" ");

	        }
	    }

	    RETURN_TRUE;
	}

为了保持这个函数的简洁，在这个函数中只有`string`类型的数组元素被输出了。你可能想知道为什么我们不用`convert_to_string()`就像我们在之前的`hello_greetme()`函数中做的那样。让我们改一下看看；把上面的for循环代码用以下的代码替换掉：

	for(zend_hash_internal_pointer_reset_ex(arr_hash, &pointer); zend_hash_get_current_data_ex(arr_hash, (void**) &data, &pointer) == SUCCESS; zend_hash_move_forward_ex(arr_hash, &pointer)) {
	    convert_to_string_ex(data);
	    PHPWRITE(Z_STRVAL_PP(data), Z_STRLEN_PP(data));
	    php_printf(" ");
	}

现在重新编译你的扩展，然后在用户空间运行如下代码：

注意这个原始的数组已经被更改了！记住，`convert_to_*()`函数的作用和调用`set_type()`是一样的。由于你在和传递进来的数组一起工作，修改它的类型将会改变原始变量。为了避免这个，你需要首选复制一份zval。为了这么做，再修改一下for循环代码如下：

	for(zend_hash_internal_pointer_reset_ex(arr_hash, &pointer); zend_hash_get_current_data_ex(arr_hash, (void**) &data, &pointer) == SUCCESS; zend_hash_move_forward_ex(arr_hash, &pointer)) {
	        zval temp;
	        temp = **data;
	        zval_copy_ctor(&temp);
	        convert_to_string(&temp);
	        PHPWRITE(Z_STRVAL(temp), Z_STRLEN(temp));
	        php_printf(" ");
	        zval_dtor(&temp);
	}
	

这个版本中最明显的部分 – `temp = **data` – 就是拷贝原始`zval`中的`data`成员，但是因为一个`zval`可能会包含附加的资源比如像`char*` 字符串，或者`HashTable*` 数组，那么这些依赖的资源需要通过`zval_copy_ctor()`来复制一份。到目前为止只有一个简单的转换，输出，最后的那个`zval_dtor()`用来释放掉`zval_copy_ctor()`拷贝的资源。

如果你想知道当我们首次介绍`convert_to_string()`的时候，为什么你不做`zval_copy_ctor()`这个工作，那是因为当传递一个变量给一个函数的时候，它会自动创建这个变量的拷贝，从而把zval从原始变量中分离开来。这个只能在基本的zval上完成，所以一些附属的资源（比如数组元素和对象属性）仍然需要在使用之前就分离。

现在你已经看到了数组的值，让我们再修改一下代码让我们也可以看到数组的key：

	for(zend_hash_internal_pointer_reset_ex(arr_hash, &pointer); zend_hash_get_current_data_ex(arr_hash, (void**) &data, &pointer) == SUCCESS; zend_hash_move_forward_ex(arr_hash, &pointer)) {

	    zval temp;
	    char *key;
	    int key_len;
	    long index;

	    if (zend_hash_get_current_key_ex(arr_hash, &key, &key_len, &index, 0, &pointer) == HASH_KEY_IS_STRING) {
	        PHPWRITE(key, key_len);
	    } else {
	        php_printf("%ld", index);
	    }

	    php_printf(" => ");
	
	    temp = **data;
	    zval_copy_ctor(&temp);
	    convert_to_string(&temp);
	    PHPWRITE(Z_STRVAL(temp), Z_STRLEN(temp));
	    php_printf(" ");
	    zval_dtor(&temp);
	}

记住数组可以有数字索引，关联字符串key，或者二者都有。调用`zend_hash_get_current_key_ex()`可以从数组当前位置来获得数组key的类型，然后用返回值来决定key的类型，可能是`HASH_KEY_IS_STRING`，`HASH_KEY_IS_LONG`，或者`HASH_KEY_NON_EXISTANT`。既然`zend_hash_get_current_data_ex()`可以返回一个`zval**`，你可以安全的假设`HASH_KEY_NON_EXISTANT`是不会被返回的，所以只有IS_STRING和IS_LONG需要被检查。

这儿有另一个迭代HashTable的方法。Zend Engine公开了三个非常相似的函数来协助这个工作：`zend_hash_apply()`，`zend_hash_apply_with_argument()`，和`zend_hash_apply_with_arguments()`。第一个就是循环一个`HashTable`，第二个允许传递一个单独的`void*`参数给它，与此同时第三个允许通过一个可变参数列表来传递数量不限的参数。hello_array_walk()显示了每个函数的使用方法：

	static int php_hello_array_walk(zval **element TSRMLS_DC)
	{
	    zval temp;
	    temp = **element;
	    zval_copy_ctor(&temp);
	    convert_to_string(&temp);
	    PHPWRITE(Z_STRVAL(temp), Z_STRLEN(temp));
	    php_printf(" ");
	    zval_dtor(&temp);
	
	    return ZEND_HASH_APPLY_KEEP;
	}

	static int php_hello_array_walk_arg(zval **element, char *greeting TSRMLS_DC)
	{
	    php_printf("%s", greeting);
	    php_hello_array_walk(element TSRMLS_CC);

	    return ZEND_HASH_APPLY_KEEP;
	}

	static int php_hello_array_walk_args(zval **element, int num_args, var_list args, zend_hash_key *hash_key)
	{
	    char *prefix = va_arg(args, char*);
	    char *suffix = va_arg(args, char*);
	    TSRMLS_FETCH();

	    php_printf("%s", prefix);
	    php_hello_array_walk(element TSRMLS_CC);
	    php_printf("%s ", suffix);

	    return ZEND_HASH_APPLY_KEEP;
	}

	PHP_FUNCTION(hello_array_walk)
	{
	    zval *zarray;
	    int print_newline = 1;

	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "a", &zarray) == FAILURE) {
	        RETURN_NULL();
	    }

	    zend_hash_apply(Z_ARRVAL_P(zarray), (apply_func_t)php_hello_array_walk TSRMLS_CC);
	    zend_hash_apply_with_argument(Z_ARRVAL_P(zarray), (apply_func_arg_t)php_hello_array_walk_arg, "Hello " TSRMLS_CC);
	    zend_hash_apply_with_arguments(Z_ARRVAL_P(zarray), (apply_func_args_t)php_hello_array_walk_args, 2, "Hello ", "Welcome to my extension!");

	    RETURN_TRUE;
	}

到现在为止你应该对与上面大部分代码相关联的函数使用很熟悉了。传递给`hello_array_walk()`的数组被循环了三次，一次没有任何参数，一次跟着一个参数，第三次跟着两个参数。在这个设计中，`walk_arg()`和`walk_args()`函数实际上依赖于没有参数的`walk()`函数，这个函数的工作是类型转换，并输出zval，显然这个工作对于这三个函数来说都是共同的。

在这个代码块中，就像在你将要使用`zend_hash_apply()`函数的大部分地方，这个`apply()`函数会返回`ZEND_HASH_APPLY_KEEP`。这个告诉`zend_hash_apply()`函数把元素留在`HashTable`中，然后继续处理下一个。在这儿还可以返回其他值：`ZEND_HASH_APPLY_REMOVE`，意思是 – 删除当前的元素，然后继续处理下一个 –`ZEND_HASH_APPLY_STOP`，意思是在当前元素处停止数组迭代，然后完全退出`zend_hash_apply()`函数。

所有组件中稍微不太熟悉的应该是`TSRMLS_FETCH()`。你可以回想一下第一部分，`TSRMLS_*`宏是线程安全资源管理层的一部分，对于保持线程之间的独立是很必要的。因为多参数版本的`zend_hash_apply()`使用了一个可变参数列表，所以这个`tsrm_ls`标示没法传递到`walk()`函数中。为了当我们在回调`php_hello_array_walk()`函数的时候可以使用线程安全机制，你需要在函数中调用`TSRMLS_FETCH()`，它会在资源池中寻找正确的线程。（注意：这个方法比直接传递参数要慢得多，所以只在无法避免的时候才用。）

用foreach这种形式的方法来迭代一个数组是很常见的任务，但是你经常会用一个数字key或者关联key在数组中查找一个特定的值。下一个函数将会根据key从一个数组中返回一个值，其中这个函数的第一个参数是这个数组，第二个参数是所需要的key。

	PHP_FUNCTION(hello_array_value)
	{
	    zval *zarray, *zoffset, **zvalue;
	    long index = 0;
	    char *key = NULL;
	    int key_len = 0;

	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "az", &zarray, &zoffset) == FAILURE) {
	        RETURN_NULL();
	    }

	    switch (Z_TYPE_P(zoffset)) {

	        case IS_NULL:
	            index = 0;
	            break;

	        case IS_DOUBLE:
	           index = (long)Z_DVAL_P(zoffset);
	            break;

	        case IS_BOOL:
	        case IS_LONG:
	        case IS_RESOURCE:
	            index = Z_LVAL_P(zoffset);
	            break;

	        case IS_STRING:
	            key = Z_STRVAL_P(zoffset);
	            key_len = Z_STRLEN_P(zoffset);
	            break;

	        case IS_ARRAY:
	            key = "Array";
	            key_len = sizeof("Array") - 1;
	            break;

	        case IS_OBJECT:
	            key = "Object";
	            key_len = sizeof("Object") - 1;
	            break;

	        default:
	            key = "Unknown";
	            key_len = sizeof("Unknown") - 1;
	    }

	    if (key && zend_hash_find(Z_ARRVAL_P(zarray), key, key_len + 1, (void**)&zvalue) == FAILURE) {
	        RETURN_NULL();
	    } else if (!key && zend_hash_index_find(Z_ARRVAL_P(zarray), index, (void**)&zvalue) == FAILURE) {
	        RETURN_NULL();
	    }

	    *return_value = **zvalue;
	    zval_copy_ctor(return_value);
	}

这个函数以一个`switch`块开始，主要是处理类型转换，方法和`Zend Engine`很像。`NULL`被当做0，`Boolean`类型被适当的转换为0或者1，`double`类型被转换成`long`(在处理过程中会被截断)，然后`resource`类型被转换成数字值。对`resource`类型的处理方式是从PHP3留下来的，当`resource`只是查询数组时候所需要的一个数字key而不是一个唯一的类型。

数组和对象被简单的当成字符串字面量“Array”或“Object”，因为转换不会有实质性结果。最后的default条件是为了能够向后兼容，当这个扩展和PHP未来的一个版本相编译的时候，这个PHP版本可能会有其他的数据类型。

如果函数正在寻找一个关联key，那么key就必须是非`NULL`，可以使用这个key的值来决定是使用关联查询还是数字索引查询。如果查询失败了，那是因为key不存在，然后函数会返回`NULL`来表明查询失败。否则`zval`会被拷贝到`return_value`中。

###符号表作为数组

如果你之前使用过`$GLOBALS`数组，你应该知道自己在PHP脚本的全局空间中声明的变量也会出现在这个数组中。回想一下，一个数组的内部实现是一个`HashTable`，一个问题出现了：“GLOBALS数组能否在一个特殊的地方被找到呢？”回答是“YES”。它存在于一个叫做`EG(symbol_table)`的Executor Globals 结构体中，`EG(symbol_table)`是一个`HashTable`（不是`HashTable*`，提醒一下你，就是`HashTable`）。

你已经知道如何在一个数组中找到关联key所对应的元素，那么现在你知道该去哪里找全局符号表，那么在扩展代码中查找变量应该是一个很容易的事情：

	PHP_FUNCTION(hello_get_global_var)
	{
	    char *varname;
	    int varname_len;
	    zval **varvalue;

	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &varname, &varname_len) == FAILURE) {
	        RETURN_NULL();
	    }

	    if (zend_hash_find(&EG(symbol_table), varname, varname_len + 1, (void**)&varvalue) == FAILURE) {
	        php_error_docref(NULL TSRMLS_CC, E_NOTICE, "Undefined variable: %s", varname);
	        RETURN_NULL();
	    }

	    *return_value = **varvalue;
	    zval_copy_ctor(return_value);
	}

现在你应该对这个很熟悉了。这个函数接收一个字符串参数，并用它在全局空间中寻找一个变量，然后返回它。

新出现的一个函数是`php_error_docref()`。你将会在PHP源码树中发现这个函数。第一个参数是一个可选的文档引用（默认情况下是当前函数）。接下来是很经常出现的`TSRMLS_CC`，之后是一个错误的严重级别，最后是一个`printf()`类型格式的字符串和一个带有错误信息实际内容的一个变量。无论你的函数什么时候出现错误，提供一些可以理解的错误信息是非常重要的。事实上，回去给`hello_array_value()`加上错误处理语句是非常好的做法。在这个教程最后的完整性检查章节也会包含这个工作。

除了全局符号表之外，Zend Engine也保留了一个局部符号表的引用。因为内部函数没有他们自己的符号表（它们为什么要有？），所以局部符号表实际上就是用来保存内部函数变量的。让我们看个简单的函数，这个函数在局部作用域设置一个变量：

	PHP_FUNCTION(hello_set_local_var)
	{
	    zval *newvar;
	    char *varname;
	    int varname_len;
	    zval *value;

	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "sz", &varname, &varname_len, &value) == FAILURE) {
	        RETURN_NULL();
	    }

	    ALLOC_INIT_ZVAL(newvar);
	    *newvar = *value;
	    zval_copy_ctor(newvar);
	    zend_hash_add(EG(active_symbol_table), varname, varname_len + 1, &newvar, sizeof(zval*), NULL);

	    RETURN_TRUE;
	}

很明显这没有什么新的东西。继续，然后把你现在手上的代码编译一下，并运行一些测试脚本。确保编译过程无误。

###引用计数

到现在为止，我们添加到`HashTable`中的`zval`不是新创建的就是新拷贝过来的。这些`zval`是独立的，拥有自己的资源，只存在于`HashTable`之中。作为一个语言设计的概念而言，这种创建，拷贝变量的方案是“足够好”的，但是我知道你很熟悉用C来编程，所以你知道如果拷贝一大块数据的话是非常耗费内存和CPU时间的，除非你遇到特殊情况不得不这么做。考虑一下这个用户空间代码块：

如果使用`zval_copy_ctor()`（实际上使用`estrndup()`来完成的）来把`$a`拷贝到`$b`，那么这个简短的脚本将会用掉8M的内存来存储两个相同的4M文件的内容。最后一步释放`$a`会使情况变得更糟，因为原始的字符串已经被`efree()`释放掉了。在C语言中完成这个应该会简单一些：`b = a; a = NULL;`

幸运的是，`Zend Engine`比以上的做法要聪明一些。当`$a`首次被创建的时候，一个隐含的string类型的zval会被创建，内容是log文件。通过调用`zend_hash_add()`来把那个`zval`分配给`$a`。当`$a`被拷贝到`$b`中的时候，可想而知，Zend Engine会做类似下面的事情：

	{
	    zval **value;
	    zend_hash_find(EG(active_symbol_table), "a", sizeof("a"), (void**)&value);

	    ZVAL_ADDREF(*value);
	    zend_hash_add(EG(active_symbol_table), "b", sizeof("b"), value, sizeof(zval*));
	}


当然，实际代码会更复杂，但是重点关注的地方是`ZVAL_ADDREF()`。记得在一个zval中有四个标准的元素。你已经看过`type`和`value`了；这次看下`refcount`。就像它的名字表示的那样，refcount指的是一个特定的`zval`在一个符号表，数组或者其他地方被引用的次数。

当你使用`ALLOC_INIT_ZVAL()`的时候，`refcount`会被设置为1，所以当你想要返回这个`zval`，或者把它加入到一个`HashTable`中的时候，不需要做任何事。在以上的代码中，你从一个`HashTable`中找到了一个`zval`，但是没有删除它，所以它的`refcount`的值符合它被引用的次数。为了在其他地方可以引用它，你需要增加它的引用计数。

当在用户空间代码中调用`unset($a)`时候，`Zend Engine`会在那个变量上执行`zval_ptr_dtor()`。使用`zval_ptr_dtor()`的重要性你是看不见的，这个调用不需要销毁这个`zval`以及它的所有内容。它实际做的事情是减少它的`refcount`。如果，我说如果，`refcount`的值为0了，那么Zend Engine会销毁这个`zval`…

原文：[http://devzone.zend.com/node/view/id/1022](http://devzone.zend.com/node/view/id/1022)