# 类的声明1

### 类声明一般是在扩展的 PHP\_MINIT_FUNCTION(name) 中
1. 在.c文件中先声明类, 一般写该行下面   
`/* True global resources - no need for thread safety here */`   

	`zend_class_entry *name_ce;`   
2. PHP_MINIT_FUNCTION 中代码:   
	 `zend_class_entry ce;`   
	 `INIT_CLASS_ENTRY(ce, "dis_ns\\dis_name", ext_methods);`      
    //INIT_CLASS_ENTRY(ce, "display_ext_name", monip_methods);    
    //宏INIT_CLASS_ENTRY展开后,实际上是给ce赋值,初始化zend_clas_entry结构体. "\\" 用来表示命名空间   
    `name_ce = zend_register_internal_class(&ce);`  
    //zend_register_internal_* 函数用来申明class(可继承 parent class), interface. 其内部将ce复制到CG(class_table)中.   
    //如果要继承父类, zend_do_inheritance 函数将会被调用, 用来copy父类的信息到本类中.   
    //zend_class_entry 中的ce_flags 用来标志类的类型, 其宏为 ZEND_ACC_*    
    //zend_class_implements 函数用来申明类的实现的接口   
	`zend_register_class_alias("ext_name", name_ce);`     
	//用来注册类别名   
3. ext_methods 为扩展的方法   
	
	


