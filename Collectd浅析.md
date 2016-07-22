CollectD 浅析
===========================

由于工作需要，最近在研究Collectd，并写了几个Python的插件，在插件Debug的过程中，
对CollectD做了一些粗浅的研究（_主要是我找不到相关细节资料说明，只能自己翻源码了_）

研究时间比较匆忙，如果有错误还望指正。


## 一. CollectD 调用浅析

### 1.1. 主要调用流程

```flow
st=>start: Start
e=>end
op_main=>operation: main()
op_init=>operation: do_init()
op_plugin_init=>operation: plugin_init_all()
op_loop=>operation: do_loop()
op_plugin=>operation: plugin_read_all()
op_update_statistics=>operation: plugin_update_internal_statistics()
op_ds_val=>operation: plugin_dispatch_values()
op_plugin_enq=>operation: plugin_write_enqueue()
op_uc_chk_t=>operation: uc_check_timeout()
op_get_iter=>operation: c_avl_get_iterator(cache_tree)
op_iter_cache_tree=>operation: c_avl_iterator_next({iter})
cond=>condition: update time out 
                Yes or No?

op_keys_list=>operation: construct a keys list

op_parse_keys=>operation: parse_identifier_vl({keys})

op_miss_dispatch=>operation: plugin_dispatch_missing({miss_val})

st->op_main->op_init->op_plugin_init()->op_loop->op_plugin->op_update_statistics->op_ds_val->op_plugin_enq->op_uc_chk_t->cond->op_get_iter->op_iter_cache_tree
cond(yes)->op_keys_list->op_parse_keys->op_miss_dispatch->op_loop
cond(no)->op_loop

```

1. `main`进入后，开始做`init`操作，即`do_init()`，其中最主要的就是`plugin_init_all()`（_初始化插件_)，包括：
    - 获得配置文件参数
    - 初始化`WriteThreads`（_没看到man文件中有这个参数，但是配置上去是有效的_）
    - 把`plugin`信息初始化到一个链表中，并注册（register）`init callback`和`read callback`
2. 进入`do_loop`后，其实就是一个无限循环，去调用插件，即`plugin_read_all()`，这个函数就干两个事情：
    - `plugin_update_internal_statistics()`函数，主要是调用`plugin_dispatch_values()`，而这个函数的功能就是调用`plugin_write_enqueue()`，将脚本获取的数据`入队`

3. 在`main`中其实还有`WriteThreads`（上图中未给出），在`WriteThreads`中有一个对应的`plugin_write_dequeue()`函数，用来把数据`出队`
    - 出队的数据传入`plugin_dispatch_values_internal()`函数
    - 径上述函数调用`fc_process_chain()`（_proc.invoke()调用_） 或者 `fc_default_action()`函数（取决与有没有使用`Chain`）
    - 然后调用`fc_bit_write_invoke()`函数
    - 最后调用`plugin_write()`函数，`这个是真正意义上的写数据的函数`




## 二. 代码片段

### 2.1. 监控采集数据的结构
```cpp
struct value_list_s
{
	value_t *values;
	int      values_len;
	cdtime_t time;
	cdtime_t interval;
	char     host[DATA_MAX_NAME_LEN];
	char     plugin[DATA_MAX_NAME_LEN];
	char     plugin_instance[DATA_MAX_NAME_LEN];
	char     type[DATA_MAX_NAME_LEN];
	char     type_instance[DATA_MAX_NAME_LEN];
	meta_data_t *meta;
};
```

### 2.2. Identifier的结构

一个`Identifier`由以下几个元素构成：
1. host
2. plugin
3. plugin instance (optional)
4. type
5. type instance (optional)

默认构成的`identifier`如下所示
> host "/" plugin ["-" plugin instance] "/" type ["-" type instance]

我们可以将其改成如下形式：

> host "." plugin ["." plugin instance] "/" type ["-" type instance]

```cpp
/*
*Filename：src/daemon/utils_cache.c
*/
int parse_identifier (char *str, char **ret_host,
		char **ret_plugin, char **ret_plugin_instance,
		char **ret_type, char **ret_type_instance)
{
	char *hostname = NULL;
	char *plugin = NULL;
	char *plugin_instance = NULL;
	char *type = NULL;
	char *type_instance = NULL;
	
	hostname = str;
	if (hostname == NULL)
		return (-1);

	//plugin = strchr (hostname, '/');
	plugin = strchr (hostname, '.');
	if (plugin == NULL)
		return (-1);
	*plugin = '\0'; plugin++;
	
	//type = strchr (plugin, '/');
	type = strrchr (plugin, '.');  // 反向查找，这样做了以后type-instance就让他 为 None
	if (type == NULL)
		return (-1);
	*type = '\0'; type++;

	//plugin_instance = strchr (plugin, '-');
	plugin_instance = strchr (plugin, '.');
	if (plugin_instance != NULL)
	{
		*plugin_instance = '\0';
		plugin_instance++;
	}

	type_instance = strchr (type, '-');
	if (type_instance != NULL)
	{
		*type_instance = '\0';
		type_instance++;
	}

	*ret_host = hostname;
	*ret_plugin = plugin;
	*ret_plugin_instance = plugin_instance;
	*ret_type = type;
	*ret_type_instance = type_instance;
	return (0);
} /* int parse_identifier */
```

```cpp
/*
* Filename：src/daemon/common.c
*/
int format_name (char *ret, int ret_len,
		const char *hostname,
		const char *plugin, const char *plugin_instance,
		const char *type, const char *type_instance)
{
  char *buffer;
  size_t buffer_size;

  buffer = ret;
  buffer_size = (size_t) ret_len;

#define APPEND(str) do {                                               \
  size_t l = strlen (str);                                             \
  if (l >= buffer_size)                                                \
    return (ENOBUFS);                                                  \
  memcpy (buffer, (str), l);                                           \
  buffer += l; buffer_size -= l;                                       \
} while (0)

  assert (plugin != NULL);
  assert (type != NULL);

  APPEND (hostname);
  //APPEND("/")
  APPEND (".");   // change "/" -> "."
  APPEND (plugin);
  if ((plugin_instance != NULL) && (plugin_instance[0] != 0))
  {
    //APPEND("/")
    APPEND (".");  // change "/" -> "."
    APPEND (plugin_instance);
  }
  APPEND ("/");
  APPEND (type);
  if ((type_instance != NULL) && (type_instance[0] != 0))
  {
    APPEND ("-");
    APPEND (type_instance);
  }
  assert (buffer_size > 0);
  buffer[0] = 0;

#undef APPEND
  return (0);
} /* int format_name */
```
