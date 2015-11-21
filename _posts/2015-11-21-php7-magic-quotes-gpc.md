---
title: PHP 7中增加Magic Quotes GPC
layout: post
---

> PHP 7 虽然没有能够如期发布，不过在明年来临之前应该能够发布正式版，由于其在功能和性能上都有很大的改进，官方生成性能是之前一个版本的两倍，因此业界的关注度和期待都很高，很多用户也准备将PHP升级到这一版本。

#### 1. MySQL模块

> 如果是从PHP 5.6升级到PHP 7则最大的变化是PHP 7中删除了MySQL模块，建议使用PDO或MySQLi进行替换，但如果代码中已经大量的使用了MySQL模块，完成代码的改造工作可能需要较长的时间。然而发现单独进行维护的MySQL模块是可以支持PHP 7的，尽管说明中已经不再建议使用，项目的地址为[https://github.com/php/pecl-database-mysql](https://github.com/php/pecl-database-mysql)。期安置方法与普通的PHP模块完全相同，即
>
```bash
phpize
./configure
make
make install
```

#### 2. Magic Quotes GPC

> PHP 5.4中删除了PHP的Magic Quotes GPC功能，即对保存用户提交数据的三个全局变量$\_GET，$\_POST，$\_COOKIE中的引号和反斜杠自动增加转义，其初衷是为了增加安全性，防止XSS和SQL注入攻击，但由于其并不能彻底的防止攻击，同时还会引入一些麻烦，在PHP 5.3中已经不再建议使用，对于防止XSS和SQL注入，建议在SQL语句中使用预处理。然而如果代码是急于使用了这一功能而进行开发的，则如果关闭这一功能会导致大量的查询错误，影响范围极大，而且不易确定具体的影响范围，修改代码会存在困难。同样出于能够快速过渡到PHP 7，这里想到的处理方法是将这个功能重新增加到PHP 7的源代码中。

#### 3. PHP 7中相关方法的改变

> 由于PHP 7是一个全新的版本，其底层做了相当大的调整，想直接将PHP 5.3中的代码简单的合并到PHP 7中显然是不能够实现的。这里简单介绍修改中涉及的一些用法上的变化，首先其涉及PHP的php\_addslashes函数，其参数类型和返回值都在PHP 7发生了改变，具体如下
>
```c
//PHP 5
PHPAPI char *php_addslashes(char *str, int length, int *new_length, int freeit TSRMLS_DC);
>
//PHP 7
PHPAPI zend_string *php_addslashes(zend_string *str, int should_free);
```

> PHP 7中定义了zend\_string数据类型，替代了原来使用的char\*，相比之下其在内存申请和释放的操作更加简单，如下是初始化一个zend\_string的方法
>
```c
zend_string* zstr = zend_string_init(*val, val_len, 0);
>
zend_string_init("magic_quotes_gpc", sizeof("magic_quotes_gpc") - 1, 0);
```

> 另外PHP 7中动态修改配置的方法zend\_alter\_ini\_entry\_ex的参数列表也发生了变化，其中主要也是将一些参数改为了zend\_string
>
```c
//PHP 5
ZEND_API int zend_alter_ini_entry_ex(char *name, uint name_length, char *new_value, uint new_value_length, int modify_type, int stage, int force_change TSRMLS_DC);
>
//PHP 7
ZEND_API int zend_alter_ini_entry_ex(zend_string *name, zend_string *new_value, int modify_type, int stage, int force_change);
```

#### 4. 需要进行的修改

> 重新增加Magic Quotes GPC的功能需要大概以下过程，首先重新添加Magic Quotes GPC相关的配置，然后在PHP注册全局变量的过程中对其进行addslashes操作，另外在对注册环境变量时临时关闭Magic Quotes GPC功能，最后在filter扩展中也增加addslashes，由于使用了此模块注册全局变量的逻辑将由这里进行处理。对PHP 7RC7的修改的patch大致如下
>
```diff
diff -Nur php-7.0.0RC7/ext/filter/filter.c php-7.0.0RC7-patched/ext/filter/filter.c
--- php-7.0.0RC7/ext/filter/filter.c	2015-11-11 19:03:35.000000000 +0800
+++ php-7.0.0RC7-patched/ext/filter/filter.c	2015-11-18 22:20:43.468976404 +0800
@@ -467,6 +467,8 @@
 		if (IF_G(default_filter) != FILTER_UNSAFE_RAW) {
 			ZVAL_STRINGL(&new_var, *val, val_len);
 			php_zval_filter(&new_var, IF_G(default_filter), IF_G(default_filter_flags), NULL, NULL, 0);
+		} else if (PG(magic_quotes_gpc) && !retval) {
+			ZVAL_NEW_STR(&new_var, php_addslashes(zend_string_init(*val, val_len, 0), 0));
 		} else {
 			ZVAL_STRINGL(&new_var, *val, val_len);
 		}
diff -Nur php-7.0.0RC7/ext/standard/basic_functions.c php-7.0.0RC7-patched/ext/standard/basic_functions.c
--- php-7.0.0RC7/ext/standard/basic_functions.c	2015-11-11 19:03:33.000000000 +0800
+++ php-7.0.0RC7-patched/ext/standard/basic_functions.c	2015-11-18 22:23:55.738978812 +0800
@@ -4619,10 +4619,7 @@
    Get the current active configuration setting of magic_quotes_gpc */
 PHP_FUNCTION(get_magic_quotes_gpc)
 {
-	if (zend_parse_parameters_none() == FAILURE) {
-		return;
-	}
-	RETURN_FALSE;
+	RETURN_LONG(PG(magic_quotes_gpc));
 }
 /* }}} */
> 
diff -Nur php-7.0.0RC7/main/main.c php-7.0.0RC7-patched/main/main.c
--- php-7.0.0RC7/main/main.c	2015-11-11 19:03:36.000000000 +0800
+++ php-7.0.0RC7-patched/main/main.c	2015-11-18 22:22:45.368977923 +0800
@@ -507,6 +507,9 @@
 	STD_PHP_INI_BOOLEAN("ignore_repeated_source",	"0",	PHP_INI_ALL,		OnUpdateBool,			ignore_repeated_source,	php_core_globals,	core_globals)
 	STD_PHP_INI_BOOLEAN("report_memleaks",		"1",		PHP_INI_ALL,		OnUpdateBool,			report_memleaks,		php_core_globals,	core_globals)
 	STD_PHP_INI_BOOLEAN("report_zend_debug",	"1",		PHP_INI_ALL,		OnUpdateBool,			report_zend_debug,		php_core_globals,	core_globals)
+	STD_PHP_INI_BOOLEAN("magic_quotes_gpc",		"1",		PHP_INI_PERDIR|PHP_INI_SYSTEM,	OnUpdateBool,	magic_quotes_gpc,		php_core_globals,	core_globals)
+	STD_PHP_INI_BOOLEAN("magic_quotes_runtime",	"0",		PHP_INI_ALL,		OnUpdateBool,			magic_quotes_runtime,	php_core_globals,	core_globals)
+	STD_PHP_INI_BOOLEAN("magic_quotes_sybase",	"0",		PHP_INI_ALL,		OnUpdateBool,			magic_quotes_sybase,	php_core_globals,	core_globals)
 	STD_PHP_INI_ENTRY("output_buffering",		"0",		PHP_INI_PERDIR|PHP_INI_SYSTEM,	OnUpdateLong,	output_buffering,		php_core_globals,	core_globals)
 	STD_PHP_INI_ENTRY("output_handler",			NULL,		PHP_INI_PERDIR|PHP_INI_SYSTEM,	OnUpdateString,	output_handler,		php_core_globals,	core_globals)
 	STD_PHP_INI_BOOLEAN("register_argc_argv",	"1",		PHP_INI_PERDIR|PHP_INI_SYSTEM,	OnUpdateBool,	register_argc_argv,		php_core_globals,	core_globals)
diff -Nur php-7.0.0RC7/main/php_globals.h php-7.0.0RC7-patched/main/php_globals.h
--- php-7.0.0RC7/main/php_globals.h	2015-11-11 19:03:36.000000000 +0800
+++ php-7.0.0RC7-patched/main/php_globals.h	2015-11-18 22:22:56.165644722 +0800
@@ -54,6 +54,10 @@
 } arg_separators;
> 
 struct _php_core_globals {
+	zend_bool magic_quotes_gpc;
+	zend_bool magic_quotes_runtime;
+	zend_bool magic_quotes_sybase;
+
 	zend_bool implicit_flush;
> 
 	zend_long output_buffering;
diff -Nur php-7.0.0RC7/main/php_variables.c php-7.0.0RC7-patched/main/php_variables.c
--- php-7.0.0RC7/main/php_variables.c	2015-11-11 19:03:36.000000000 +0800
+++ php-7.0.0RC7-patched/main/php_variables.c	2015-11-18 22:19:14.128975284 +0800
@@ -49,7 +49,11 @@
 	assert(strval != NULL);
> 
 	/* Prepare value */
-	ZVAL_NEW_STR(&new_entry, zend_string_init(strval, str_len, 0));
+	if (PG(magic_quotes_gpc)) {
+		ZVAL_NEW_STR(&new_entry, php_addslashes(zend_string_init(strval, str_len, 0), 0));
+	} else {
+		ZVAL_NEW_STR(&new_entry, zend_string_init(strval, str_len, 0));
+	}
 	php_register_variable_ex(var, &new_entry, track_vars_array);
 }
> 
@@ -57,7 +61,7 @@
 {
 	char *p = NULL;
 	char *ip = NULL;		/* index pointer */
-	char *index;
+	char *index, *escaped_index = NULL;
 	char *var, *var_orig;
 	size_t var_len, index_len;
 	zval gpc_element, *gpc_element_p;
@@ -180,11 +184,18 @@
 					return;
 				}
 			} else {
-				gpc_element_p = zend_symtable_str_find(symtable1, index, index_len);
+				if (PG(magic_quotes_gpc)) {
+					zend_string* zstr = php_addslashes(zend_string_init(index, index_len, 0), 0);
+					escaped_index = ZSTR_VAL(zstr);
+					index_len = ZSTR_LEN(zstr);
+				} else {
+					escaped_index = index;
+				}
+				gpc_element_p = zend_symtable_str_find(symtable1, escaped_index, index_len);
 				if (!gpc_element_p) {
 					zval tmp;
 					array_init(&tmp);
-					gpc_element_p = zend_symtable_str_update_ind(symtable1, index, index_len, &tmp);
+					gpc_element_p = zend_symtable_str_update_ind(symtable1, escaped_index, index_len, &tmp);
 				} else {
 					if (Z_TYPE_P(gpc_element_p) == IS_INDIRECT) {
 						gpc_element_p = Z_INDIRECT_P(gpc_element_p);
@@ -216,6 +227,13 @@
 				zval_ptr_dtor(&gpc_element);
 			}
 		} else {
+			if (PG(magic_quotes_gpc)) {
+				zend_string* zstr = php_addslashes(zend_string_init(index, index_len, 0), 0);
+				escaped_index = ZSTR_VAL(zstr);
+				index_len = ZSTR_LEN(zstr);
+			} else {
+				escaped_index = index;
+			}
 			/*
 			 * According to rfc2965, more specific paths are listed above the less specific ones.
 			 * If we encounter a duplicate cookie name, we should skip it, since it is not possible
@@ -224,10 +242,10 @@
 			 */
 			if (Z_TYPE(PG(http_globals)[TRACK_VARS_COOKIE]) != IS_UNDEF &&
 				symtable1 == Z_ARRVAL(PG(http_globals)[TRACK_VARS_COOKIE]) &&
-				zend_symtable_str_exists(symtable1, index, index_len)) {
+				zend_symtable_str_exists(symtable1, escaped_index, index_len)) {
 				zval_ptr_dtor(&gpc_element);
 			} else {
-				gpc_element_p = zend_symtable_str_update_ind(symtable1, index, index_len, &gpc_element);
+				gpc_element_p = zend_symtable_str_update_ind(symtable1, escaped_index, index_len, &gpc_element);
 			}
 		}
 	}
@@ -496,6 +514,13 @@
 	char **env, *p, *t = buf;
 	size_t alloc_size = sizeof(buf);
 	unsigned long nlen; /* ptrdiff_t is not portable */
+	
+	/* turn off magic_quotes while importing environment variables */
+	int magic_quotes_gpc = PG(magic_quotes_gpc);
+
+	if (magic_quotes_gpc) {
+		zend_alter_ini_entry_ex(zend_string_init("magic_quotes_gpc", sizeof("magic_quotes_gpc") - 1, 0), zend_string_init("0", 1, 0), ZEND_INI_SYSTEM, ZEND_INI_STAGE_ACTIVATE, 1);
+	}
> 
 	for (env = environ; env != NULL && *env != NULL; env++) {
 		p = strchr(*env, '=');
@@ -514,6 +539,10 @@
 	if (t != buf && t != NULL) {
 		efree(t);
 	}
+	
+	if (magic_quotes_gpc) {
+		zend_alter_ini_entry_ex(zend_string_init("magic_quotes_gpc", sizeof("magic_quotes_gpc") - 1, 0), zend_string_init("1", 1, 0), ZEND_INI_SYSTEM, ZEND_INI_STAGE_ACTIVATE, 1);
+	}
 }
> 
 zend_bool php_std_auto_global_callback(char *name, uint name_len)
@@ -593,9 +622,14 @@
 static inline void php_register_server_variables(void)
 {
 	zval request_time_float, request_time_long;
+	/* turn off magic_quotes while importing server variables */
+	int magic_quotes_gpc = PG(magic_quotes_gpc);
> 
 	zval_ptr_dtor(&PG(http_globals)[TRACK_VARS_SERVER]);
 	array_init(&PG(http_globals)[TRACK_VARS_SERVER]);
+	if (magic_quotes_gpc) {
+		zend_alter_ini_entry_ex(zend_string_init("magic_quotes_gpc", sizeof("magic_quotes_gpc") - 1, 0), zend_string_init("0", 1, 0), ZEND_INI_SYSTEM, ZEND_INI_STAGE_ACTIVATE, 1);
+	}
> 
 	/* Server variables */
 	if (sapi_module.register_server_variables) {
@@ -618,6 +652,10 @@
 	php_register_variable_ex("REQUEST_TIME_FLOAT", &request_time_float, &PG(http_globals)[TRACK_VARS_SERVER]);
 	ZVAL_LONG(&request_time_long, zend_dval_to_lval(Z_DVAL(request_time_float)));
 	php_register_variable_ex("REQUEST_TIME", &request_time_long, &PG(http_globals)[TRACK_VARS_SERVER]);
+
+	if (magic_quotes_gpc) {
+		zend_alter_ini_entry_ex(zend_string_init("magic_quotes_gpc", sizeof("magic_quotes_gpc") - 1, 0), zend_string_init("1", 1, 0), ZEND_INI_SYSTEM, ZEND_INI_STAGE_ACTIVATE, 1);
+	}
 }
 /* }}} */
>
diff -Nur php-7.0.0RC7/sapi/cgi/cgi_main.c php-7.0.0RC7-patched/sapi/cgi/cgi_main.c
--- php-7.0.0RC7/sapi/cgi/cgi_main.c	2015-11-11 19:03:24.000000000 +0800
+++ php-7.0.0RC7-patched/sapi/cgi/cgi_main.c	2015-11-18 22:19:40.342308948 +0800
@@ -632,8 +632,16 @@
 	php_php_import_environment_variables(array_ptr);
> 
 	if (fcgi_is_fastcgi()) {
+		int magic_quotes_gpc = PG(magic_quotes_gpc);
 		fcgi_request *request = (fcgi_request*) SG(server_context);
+
+		if (magic_quotes_gpc) {
+			zend_alter_ini_entry_ex(zend_string_init("magic_quotes_gpc", sizeof("magic_quotes_gpc") - 1, 0), zend_string_init("0", 1, 0), ZEND_INI_SYSTEM, ZEND_INI_STAGE_ACTIVATE, 1);
+		}
 		fcgi_loadenv(request, cgi_php_load_env_var, array_ptr);
+		if (magic_quotes_gpc) {
+			zend_alter_ini_entry_ex(zend_string_init("magic_quotes_gpc", sizeof("magic_quotes_gpc") - 1, 0), zend_string_init("1", 1, 0), ZEND_INI_SYSTEM, ZEND_INI_STAGE_ACTIVATE, 1);
+		}
 	}
 }
> 
diff -Nur php-7.0.0RC7/sapi/fpm/fpm/fpm_main.c php-7.0.0RC7-patched/sapi/fpm/fpm/fpm_main.c
--- php-7.0.0RC7/sapi/fpm/fpm/fpm_main.c	2015-11-11 19:03:23.000000000 +0800
+++ php-7.0.0RC7-patched/sapi/fpm/fpm/fpm_main.c	2015-11-18 22:20:06.398975944 +0800
@@ -562,6 +562,7 @@
 void cgi_php_import_environment_variables(zval *array_ptr) /* {% raw %}{{{ {% endraw %}*/
 {
 	fcgi_request *request = NULL;
+	int magic_quotes_gpc;
> 
 	if (Z_TYPE(PG(http_globals)[TRACK_VARS_ENV]) == IS_ARRAY &&
 		Z_ARR_P(array_ptr) != Z_ARR(PG(http_globals)[TRACK_VARS_ENV]) &&
@@ -583,7 +584,14 @@
 	php_php_import_environment_variables(array_ptr);
> 
 	request = (fcgi_request*) SG(server_context);
+	magic_quotes_gpc = PG(magic_quotes_gpc);
+	if (magic_quotes_gpc) {
+		zend_alter_ini_entry_ex(zend_string_init("magic_quotes_gpc", sizeof("magic_quotes_gpc") - 1, 0), zend_string_init("0", 1, 0), ZEND_INI_SYSTEM, ZEND_INI_STAGE_ACTIVATE, 1);
+	}
 	fcgi_loadenv(request, cgi_php_load_env_var, array_ptr);
+	if (magic_quotes_gpc) {
+		zend_alter_ini_entry_ex(zend_string_init("magic_quotes_gpc", sizeof("magic_quotes_gpc") - 1, 0), zend_string_init("1", 1, 0), ZEND_INI_SYSTEM, ZEND_INI_STAGE_ACTIVATE, 1);
+	}
 }
 /* }}} */
```

#### 5. 修改后的效果

> 通过patch的方式将以上patch合并到PHP源代码中，并编译PHP，启动后执行以下代码，即http://127.0.0.1:8088/?v%22=%27\1%27
>
```php
<?php
print_r($_GET);
print_r(PHP_VERSION);
```

> 会得到以下输出
>
```
Array ( [v\"] => \'\\1\' ) 7.0.0RC7
```

> 可以看到其自动对单引号、双引号、反斜杠都增加了转义，实现了Magic Quotes GPC的作用。虽然官方已经不再建议用这个功能，但为了能够尽快的升级PHP 7，如果一直等待代码的修改完成恐怕就要等到PHP 8了。
