---
layout: post
title: PHP扩展编写第二步：参数，数组，以及ZVAL「续」
city: 南京
tags: [translate]
---

###拷贝 VS 引用

这有两个方法来引用一个`zval`。第一种，上面介绍过的，叫做**copy-on-write referencing**。第二种，叫做**full referencing**，就是用户空间脚本编写者非常熟悉的「引用」关系，当写出如下代码：`$a = &$b;`的时候就会发生。
在一个zval中，这两种类型通过成员`is_ref`区分开来，当`is_ref`为0的时候是拷贝引用（copy references），非0的时候就是完全引用（full references）。注意一个`zval`不可能既是拷贝引用，又是完全引用。所以如果一个变量刚开始`is_ref`值非0，然后作为一个拷贝被赋给了一个新变量，那么肯定会执行一个完全拷贝。考虑下面的用户空间代码：

	<?php
	    $a = 1;
	    $b = &$a;
	    $c = $a;
	?>

在这段代码中，为$a变量创建了一个`zval`，初始化`is_ref`为0，`refcount`为1。当`$a`以引用的方式赋给了`$b`，其`is_ref`变成了1，然后`refcount`增加到2。当把$a赋给$c的时候，Zend Engine不能简单的把`refcount`增加到3，因为`$c`将被看做是`$a`的完全引用。关闭`is_ref`也不会起作用，因为如果这样做的话`$b`就是`$a`的一个拷贝了，而不是一个引用了。所以这个时候一个新的`zval`被分配了，然后通过`zval_copy_ctor()`来把原始的值拷贝进来。原始的zval现在是`is_ref==1`，`refcount==2`，而新的`zval`中`is_ref=0`，`refcount=1`。现在让我们稍微把代码顺序调整一下再看：

	<?php
	    $a = 1;
	    $c = $a;
	    $b = &$a;
	?>


最终的结果是一样的，`$b`是`$a`的完全引用，`$c`是`$a`的一个拷贝。这次，内部的行为稍微有些不同。像之前一样，在开始的时候为`$a`创建了一个新的`zval`，然后设置`is_ref==0`，`refcount=1`。之后，`$c = $a;`这个语句把上面那个相同的`zval`赋给`$c`变量，同时增加`refcount`到2，`is_ref`仍然是0。当Zend Engine遇到`$b = &amp;$a;`的时候，它只是想要设置is_ref为1，但是不能这么做因为这么做会影响`$c`。替代的做法是它创建一个新的`zval`，通过zval_copy_ctor()来把原始的内容拷贝进来，然后对原始`zval`的`refcount`减一从而来指明$a不在使用那个`zval`了。它把新`zval`的`is_ref`设置为1，`refcount`设置为2，然后更新`$a`和`$b`变量，让其引用这个`zval`。

###完整性检查

像之前一样，我们三个主要文件中的完整代码已经列在下面了：

###config.m4

	PHP_ARG_ENABLE(hello, [whether to enable Hello World support],
	[ --enable-hello   Enable Hello World support])

	if test "$PHP_HELLO" = "yes"; then
	  AC_DEFINE(HAVE_HELLO, 1, [Whether you have Hello World])
	  PHP_NEW_EXTENSION(hello, hello.c, $ext_shared)
	fi

###php_hello.h

	#ifndef PHP_HELLO_H
	#define PHP_HELLO_H 1

	#ifdef ZTS
	#include "TSRM.h"
	#endif

	ZEND_BEGIN_MODULE_GLOBALS(hello)
	    long counter;
	    zend_bool direction;
	ZEND_END_MODULE_GLOBALS(hello)

	#ifdef ZTS
	#define HELLO_G(v) TSRMG(hello_globals_id, zend_hello_globals *, v)
	#else

	#define HELLO_G(v) (hello_globals.v)

	#endif

	#define PHP_HELLO_WORLD_VERSION "1.0"

	#define PHP_HELLO_WORLD_EXTNAME "hello"

	PHP_MINIT_FUNCTION(hello);
	PHP_MSHUTDOWN_FUNCTION(hello);
	PHP_RINIT_FUNCTION(hello);

	PHP_FUNCTION(hello_world);
	PHP_FUNCTION(hello_long);
	PHP_FUNCTION(hello_double);

	PHP_FUNCTION(hello_bool);
	PHP_FUNCTION(hello_null);
	PHP_FUNCTION(hello_greetme);

	PHP_FUNCTION(hello_add);
	PHP_FUNCTION(hello_dump);
	PHP_FUNCTION(hello_array);

	PHP_FUNCTION(hello_array_strings);
	PHP_FUNCTION(hello_array_walk);
	PHP_FUNCTION(hello_array_value);
	PHP_FUNCTION(hello_get_global_var);
	PHP_FUNCTION(hello_set_local_var);

	extern zend_module_entry hello_module_entry;

	#define phpext_hello_ptr &hello_module_entry

	#endif

###hello.c

	#ifdef HAVE_CONFIG_H
	#include "config.h"
	#endif

 
	#include "php.h"
	#include "php_ini.h"
	#include "php_hello.h"

	ZEND_DECLARE_MODULE_GLOBALS(hello)

	static function_entry hello_functions[] = {
	    PHP_FE(hello_world, NULL)
	    PHP_FE(hello_long, NULL)
	    PHP_FE(hello_double, NULL)
	    PHP_FE(hello_bool, NULL)
	    PHP_FE(hello_null, NULL)
	    PHP_FE(hello_greetme, NULL)
	    PHP_FE(hello_add, NULL)
	    PHP_FE(hello_dump, NULL)
	    PHP_FE(hello_array, NULL)
	    PHP_FE(hello_array_strings, NULL)
	    PHP_FE(hello_array_walk, NULL)
	    PHP_FE(hello_array_value, NULL)
	    PHP_FE(hello_get_global_var, NULL)
	    PHP_FE(hello_set_local_var, NULL)
	    {NULL, NULL, NULL}
	};

	zend_module_entry hello_module_entry = {

	#if ZEND_MODULE_API_NO >= 20010901
	    STANDARD_MODULE_HEADER,
	#endif

	    PHP_HELLO_WORLD_EXTNAME,
	    hello_functions,
	    PHP_MINIT(hello),
	    PHP_MSHUTDOWN(hello),
	    PHP_RINIT(hello),
	    NULL,
	    NULL,
	#if ZEND_MODULE_API_NO >= 20010901
	    PHP_HELLO_WORLD_VERSION,
	#endif
	    STANDARD_MODULE_PROPERTIES

	};

	#ifdef COMPILE_DL_HELLO
	    ZEND_GET_MODULE(hello)
	#endif

	PHP_INI_BEGIN()
	    PHP_INI_ENTRY("hello.greeting", "Hello World", PHP_INI_ALL, NULL)
	    STD_PHP_INI_ENTRY("hello.direction", "1", PHP_INI_ALL, OnUpdateBool, direction, zend_hello_globals, hello_globals)
	PHP_INI_END()

	static void php_hello_init_globals(zend_hello_globals *hello_globals)
	{
	    hello_globals->direction = 1;
	}

	PHP_RINIT_FUNCTION(hello)
	{
	    HELLO_G(counter) = 0;
	    return SUCCESS;
	}

	PHP_MINIT_FUNCTION(hello)
	{
	    ZEND_INIT_MODULE_GLOBALS(hello, php_hello_init_globals, NULL);
	    REGISTER_INI_ENTRIES();
	    return SUCCESS;
	}

	PHP_MSHUTDOWN_FUNCTION(hello)
	{
	    UNREGISTER_INI_ENTRIES();
	    return SUCCESS;
	}

	PHP_FUNCTION(hello_world)
	{
	    RETURN_STRING("Hello World", 1);
	}

	PHP_FUNCTION(hello_long)
	{
	    if (HELLO_G(direction)) {
	        HELLO_G(counter)++;
	    } else {
	        HELLO_G(counter)--;
	    }
	
	    RETURN_LONG(HELLO_G(counter));
	}

	PHP_FUNCTION(hello_double)
	{
	    RETURN_DOUBLE(3.1415926535);
	}

	PHP_FUNCTION(hello_bool)
	{
	    RETURN_BOOL(1);
	}

	PHP_FUNCTION(hello_null)
	{
	    RETURN_NULL();
	}

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

	PHP_FUNCTION(hello_dump)
	{
	    zval *uservar;

	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", &uservar) == FAILURE) {
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
	    php_printf("mysubarray->refcount = %d ", mysubarray->refcount);
	    mysubarray->refcount = 2;
	    php_printf("mysubarray->refcount = %d ", mysubarray->refcount);
	    add_assoc_zval(return_value, "subarray", mysubarray);
	    php_printf("mysubarray->refcount = %d ", mysubarray->refcount);

	}

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
	
	    RETURN_TRUE;
	}

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
	    zend_hash_internal_pointer_reset(Z_ARRVAL_P(zarray));
	    zend_hash_apply_with_argument(Z_ARRVAL_P(zarray), (apply_func_arg_t)php_hello_array_walk_arg, "Hello " TSRMLS_CC);
	    zend_hash_apply_with_arguments(Z_ARRVAL_P(zarray), (apply_func_args_t)php_hello_array_walk_args, 2, "Hello ", "Welcome to my extension!");

	    RETURN_TRUE;
	}

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
	        php_error_docref(NULL TSRMLS_CC, E_NOTICE, "Undefined index: %s", key);
	        RETURN_NULL();

	    } else if (!key && zend_hash_index_find(Z_ARRVAL_P(zarray), index, (void**)&zvalue) == FAILURE) {
	        php_error_docref(NULL TSRMLS_CC, E_NOTICE, "Undefined index: %ld", index);
	        RETURN_NULL();
	    }

	    *return_value = **zvalue;
	    zval_copy_ctor(return_value);
	}

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

Gist:

* config.m4:    [https://gist.github.com/2843439](https://gist.github.com/2843439)
* php\_hello.h: [https://gist.github.com/2843449](https://gist.github.com/2843449)
* hello.c:      [https://gist.github.com/2843458](https://gist.github.com/2843458)

下一步是什么？

在这个扩展编写系列教程的第二部分中，你学到了如何接收函数参数，你创建和使用了数组，然后，最重要的，你看了`zval`的内部工作过程。在第三部分，你将要看到`resource`数据类型，并且和更加复杂的数据结构打交道。

原文：[http://devzone.zend.com/node/view/id/1023](http://devzone.zend.com/node/view/id/1023)
