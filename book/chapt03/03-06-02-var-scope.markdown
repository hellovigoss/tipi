# 变量的作用域

作用域是变量一个很重要的概念。变量的作用域就是语句的一个作用范围，在这个范围内变量为可见的。
换句话说，变量的作用域是可以访问该变量的代码区域。我们常见的包含作用域概念的变量包括全局变量和局部变量。
变量的作用域与变量的生命周期有一定的联系，如在一个函数中定义的变量，这个变量的作用域从变量声明的时候开始到这个函数结束的时候。
这种变量我们称之为局部变量。它的生命周期开始于函数开始，结束于函数的调用完成之时。
对于不同作用域的变量，如果存在冲突情况，如全局变量中有一个名为$a的变量，在局部变量中也存在一个名为$a的变量，
此时如何区分呢？这个也是我们将要探讨的问题。

对于全局变量，ZEND内核有一个_zend_executor_globals结构，该结构中的symbol_table就是全局符号表，
其中保存了在顶层作用域中的变量。同样，函数或者对象的方法在被调用时会创建active_symbol_table来保存局部变量。
当程序在顶层中使用某个变量时，ZE就会在symbol_table中进行遍历，同理，每个函数也会有对应的active_symbol_table来供程序使用。

对于局部变量，如我们调用的一个函数中的变量，ZE使用_zend_execute_data来存储
某个单独的op_array（每个函数都会生成单独的op_array)执行过程中所需要的信息，它的结构如下：

	[c]
	struct _zend_execute_data {
		struct _zend_op *opline;
		zend_function_state function_state;
		zend_function *fbc; /* Function Being Called */
		zend_class_entry *called_scope;
		zend_op_array *op_array;
		zval *object;
		union _temp_variable *Ts;
		zval ***CVs;
		HashTable *symbol_table;
		struct _zend_execute_data *prev_execute_data;
		zval *old_error_reporting;
		zend_bool nested;
		zval **original_return_value;
		zend_class_entry *current_scope;
		zend_class_entry *current_called_scope;
		zval *current_this;
		zval *current_object;
		struct _zend_op *call_opline;
	};

函数中的局部变量就存储在_zend_execute_data的symbol_table中，在执行当前函数的op_array时，
全局zend_executor_globals中的*active_symbol_table会指向当前_zend_execute_data中的*symbol_table。
而此时，其他函数中的symbol_table不会出现在当前的active_symbol_table中，如此便实现了局部变量。
相关操作在 Zend/zend_vm_execute.h 文件中定义的execute函数中一目了然，如下所示代码：

    [c]
    zend_vm_enter:
	/* Initialize execute_data */
	execute_data = (zend_execute_data *)zend_vm_stack_alloc(
		sizeof(zend_execute_data) +
		sizeof(zval**) * op_array->last_var * (EG(active_symbol_table) ? 1 : 2) +
		sizeof(temp_variable) * op_array->T TSRMLS_CC);

    EX(symbol_table) = EG(active_symbol_table);
	EX(prev_execute_data) = EG(current_execute_data);
	EG(current_execute_data) = execute_data;

EX宏的作用是取结构体zend_execute_data的字段值，如下所示代码：

    [c]
    #define EX(element) execute_data->element

所以，变量的作用域是使用不同的符号表来实现的，于是顶层的全局变量在函数内部使用时，需要先使用上一节中提到的global语句进行变量的跨域操作。