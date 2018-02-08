## SAPI
[SAPI](https://en.wikipedia.org/wiki/Server_Application_Programming_Interface)是一套供开发者扩展web server能力的接口规范. 在PHP中, 提供了几种不同的SAPI, 用来满足接入不同的web server. 如apache, nigix等.  
目前在PHP源码中(sapi目录下)可以看到有apache2handler, cgi, cli, embed, fpm, phpdgb 等方式.  

在PHP内部中, 每个sapi都需定义一个 sapi_module_struct 数据结构, 与php内核交互, 实现php代码的执行, 得到返回结果.  

#### sapi_module_struct 结构体

```
struct _sapi_module_struct {
	char *name;
	char *pretty_name;

	int (*startup)(struct _sapi_module_struct *sapi_module);
	int (*shutdown)(struct _sapi_module_struct *sapi_module);

	int (*activate)(void);
	int (*deactivate)(void);

	size_t (*ub_write)(const char *str, size_t str_length);
	void (*flush)(void *server_context);
	zend_stat_t *(*get_stat)(void);
	char *(*getenv)(char *name, size_t name_len);

	void (*sapi_error)(int type, const char *error_msg, ...) ZEND_ATTRIBUTE_FORMAT(printf, 2, 3);

	int (*header_handler)(sapi_header_struct *sapi_header, sapi_header_op_enum op, sapi_headers_struct *sapi_headers);
	int (*send_headers)(sapi_headers_struct *sapi_headers);
	void (*send_header)(sapi_header_struct *sapi_header, void *server_context);

	size_t (*read_post)(char *buffer, size_t count_bytes);
	char *(*read_cookies)(void);

	void (*register_server_variables)(zval *track_vars_array);
	void (*log_message)(char *message, int syslog_type_int);
	double (*get_request_time)(void);
	void (*terminate_process)(void);

	char *php_ini_path_override;

	void (*default_post_reader)(void);
	void (*treat_data)(int arg, char *str, zval *destArray);
	char *executable_location;

	int php_ini_ignore;
	int php_ini_ignore_cwd; /* don't look for php.ini in the current directory */

	int (*get_fd)(int *fd);

	int (*force_http_10)(void);

	int (*get_target_uid)(uid_t *);
	int (*get_target_gid)(gid_t *);

	unsigned int (*input_filter)(int arg, char *var, char **val, size_t val_len, size_t *new_val_len);

	void (*ini_defaults)(HashTable *configuration_hash);
	int phpinfo_as_text;

	char *ini_entries;
	const zend_function_entry *additional_functions;
	unsigned int (*input_filter_init)(void);
};
```  

### 一个SAPI的流程
以cli sapi为例, 大致先了解php的启动执行流程  

1. 在 sapi/cli/php_cli.c 文件中可以找到 main 函数, 即在命令行中输入 ./php 执行入口.  
可以看到 cli sapi 的 sapi_module_struct 默认值为 cli_sapi_module, 它指定了 sapi_module_struct 一部分成员.  

```
static sapi_module_struct cli_sapi_module = {
	"cli",							/* name */
	"Command Line Interface",    	/* pretty name */

	php_cli_startup,				/* startup */
	php_module_shutdown_wrapper,	/* shutdown */

	NULL,							/* activate */
	sapi_cli_deactivate,			/* deactivate */

	sapi_cli_ub_write,		    	/* unbuffered write */
	sapi_cli_flush,				    /* flush */
	NULL,							/* get uid */
	NULL,							/* getenv */

	php_error,						/* error handler */

	sapi_cli_header_handler,		/* header handler */
	sapi_cli_send_headers,			/* send headers handler */
	sapi_cli_send_header,			/* send header handler */

	NULL,				            /* read POST data */
	sapi_cli_read_cookies,          /* read Cookies */

	sapi_cli_register_variables,	/* register server variables */
	sapi_cli_log_message,			/* Log message */
	NULL,							/* Get request time */
	NULL,							/* Child terminate */

	STANDARD_SAPI_MODULE_PROPERTIES
};
```
`cli_sapi_module.additional_functions = additional_functions;`  

2. 然后是调用的了信号处理函数 zend_signal_startup, 将 SIGILL, SIGABRT, SIGFPE, SIGKILL, SIGSEGV, SIGCONT, SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU, SIGBUS, SIGSYS, SIGTRAP 外的信号量预处理, 将信号量的默认句柄函数收集到 global_orig_handlers 数组中, 供后续使用  

3. 解析部分命令行参数, 如 -c  PHP配置文件php.ini等.  

4. sapi_startup: 
初始化一个 sapi\_globals\_struct `sapi\_globals` 的全局变量, 用来存放http的头信息. 宏SG(v),是对该变量的取值的简化.  

```
typedef struct _sapi_globals_struct {
	void *server_context;
	sapi_request_info request_info;
	sapi_headers_struct sapi_headers;
	int64_t read_post_bytes;
	unsigned char post_read;
	unsigned char headers_sent;
	zend_stat_t global_stat;
	char *default_mimetype;
	char *default_charset;
	HashTable *rfc1867_uploaded_files;
	zend_long post_max_size;
	int options;
	zend_bool sapi_started;
	double global_request_time;
	HashTable known_post_content_types;
	zval callback_func;
	zend_fcall_info_cache fci_cache;
} sapi_globals_struct;  
```

5. sapi\_module->startup(), 在CLI中, 执行的为 php\_cli\_startup 函数. 而php\_cli\_startup 直接调用的php内核函数 php\_module\_startup. 
初始化 SG(sapi\_headers) 成员, 然后调用sapi\_moudle的 activate 函数和 input\_filter\_init 函数. (如果被定义, 而cli并没有实现这两个函数)  

php\_output\_startup 初始化了php输出缓冲的全局变量, 宏OG(v) 用来简化取值.  

``` 
ZEND_BEGIN_MODULE_GLOBALS(output)
	zend_stack handlers;
	php_output_handler *active;
	php_output_handler *running;
	const char *output_start_filename;
	int output_start_lineno;
	int flags;
ZEND_END_MODULE_GLOBALS(output)
 ```

指定php\_output\_direct(函数指针) 为php\_output\_stdout, 即向终端stdout输出.  

php\_startup\_ticks 初始化了PG(tick\_functions)变量, 宏PG(v)是对php\_core\_globals `core\_globals` 全局变量的取值, 定义在 main/php_globals.h 中, 从取名上就可以看, 这是PHP一个很核心的全局变量. 一些对PHP的状态设置等全部包含这里.  

gc\_globals\_ctor 初始化GC的全局变量 gc_globals.  

设置zend\_startup函数需要的参数 zend\_utility\_functions, 一个工具集的结构体, 包含错误回调, 输出格式化等函数的变量.  
zend_startup 函数:
  * start_memory_manager 初始换内存变量 alloc_globals    
  * virtual_cwd_startup 初始化虚机路径变量 cwd_globals  
  * zend_startup_extensions_mechanism 初始化了 zend_extensions 列表  
  * 给zend中的一些全局变量函数指针赋值, 如 zend_error_cb, zend_write 等.  
  * zend_init_opcodes_handlers 初始化了opcode对应的句柄函数  
  * 初始化了CG(function_table), CG(class_table), CG(auto_globals), EG(zend_constants) 4个hash变量. 宏CG(v)是对zend_compiler_globals compiler_globals的取值, 宏EG(v)是对zend_executor_globals executor_globals 的取值.  
  * 初始化了 module_registry hash变量  
  * zend_init_rsrc_list_dtors 函数初始化了 list_destructors hash变量  
  * ini_scanner_globals_ctor 函数中 memset 变量 ini_scanner_globals  
  * php_scanner_globals_ctor 函数中 memset 变量 language_scanner_globals
  * zend_set_default_compile_time_values 指定了 CG(short_tags) = 1, CG(compiler_options) = 1<<1  
  * EG(error_reporting) = E_ALL & ~E_NOTICE;  
  * zend_interned_strings_init 函数初始化了 interned string  
  * zend_startup_builtin_functions 函数设置了 EG(current_module)  
  * zend_register_standard_constants 函数设置了一些PHP用到的常量, 如报错级别: E_ERROR, TRUE, FALSE, NULL等, 保存在 EG(zend_constants) 中.  
  * 注册GLOBALS变量到 CG(auto_globals) 中.  
  * zend_ini_startup 初始化 EG(ini_directives) 变量 (hashtable)  

6. setlocale, tzset  
7. 注册PHP相关系统常量, 如: PHP的版本, OS, INI相关  
8. 注册output相关的常量, 如: PHP_OUTPUT_HANDLER_START 等  
9. 注册rfc1867,上传相关的常量, 如: UPLOAD_ERR_OK, UPLOAD_ERR_FORM_SIZE 等  

10. php_init_config 函数 (TODO)  

11. php_startup_auto_globals 注册\_GET, \_POST, \_COOKIE, \_SERVER, \_ENV, \_REQUEST, \_FILES等变量到 CG(auto_globals) 中.  

12. zend_set_utility_values (TODO)  

13. php_startup_sapi_content_types 函数, 设置 sapi_module的 default_post_reader, treat_data, input_filter, input_filter_init 的默认指向函数  

13. php_register_internal_extensions_func 函数, 注册内置模块, basic, bcmath(编译指定), canendar, ftp, mbstring(编译指定), pcre, session; 变量module_registry保存注册的模块, 如果这些模块有函数列表 (functions), 那么将其函数列表加入到 CG(function_table) 中.  

14. php_ini_register_extensions 函数, 加载php.ini 中指定的 zend扩展 (zend_extensions) 和 php扩展 (module_registry)  

15. zend_startup_modules 函数, 将 module_registry 中的扩展依次执行zend_startup_module_ex:  
* 如果模块指定了globals_ctor, 则先执行 globals_ctor 函数  
* 如果模块指定了module_startup_func, 则执行 module_startup_func 函数  

16. zend_startup_extensions 函数, 将 zend_extensions 中的扩展依次执行 zend_extension_startup:  
* 如果扩展指定了startup, 则执行 startup 函数  

17. 如果sapi 定义了 additional_functions 函数列表, 则将这些函数注册到 CG(function_table) 中.  

18. php_disable_functions 函数, 将php.ini 文件中指定的不可用函数, PG(disable_functions), 标记为 display_disabled_function, 修改func的handler 句柄  

19. php_disable_classes 函数, 将php.ini 文件中指定的不可用类, PG(disable_classes), 标记为 display_disabled_class. 修改 class 的 create_object 句柄.  








