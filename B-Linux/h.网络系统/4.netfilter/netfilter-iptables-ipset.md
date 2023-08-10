1.netfilter

2.iptables
```cpp
struct netns_xt {
	struct list_head tables[NFPROTO_NUMPROTO]; /* 挂载各个协议的xt_table */
	bool notrack_deprecated_warning;
#if defined(CONFIG_BRIDGE_NF_EBTABLES) || defined(CONFIG_BRIDGE_NF_EBTABLES_MODULE)
	struct ebt_table *broute_table;
	struct ebt_table *frame_filter;
	struct ebt_table *frame_nat;
#endif
};

struct xt_table {
	struct list_head list;

	/* hook掩码,表示该类表都挂载在哪个hook点 */
	unsigned int valid_hooks;

	/* 表中匹配项的指针 */
	struct xt_table_info *private;

	/* Set this to THIS_MODULE if you are a module, otherwise NULL */
	struct module *me;

	u_int8_t af;		/* address/protocol family */
	int priority;		/* hook order */

	/* A unique name... */
	const char name[XT_TABLE_MAXNAMELEN];
};

/* 该结构用于创建xt_table_info和配置切换的时候使用 */
struct ipt_replace {
	/* 表名，eg：filter/nat/raw/mangle */
	char name[XT_TABLE_MAXNAMELEN];

	/* hook掩码，标识该表所在哪个hook点 */
	unsigned int valid_hooks;

	/* 表示要添加到表中的新规则的数量 */
	unsigned int num_entries;

	/* 表示加入表中新规则的总大小 */
	unsigned int size;

	/* hook首规则偏移 */
	unsigned int hook_entry[NF_INET_NUMHOOKS];

	/* hook尾规则的偏移. */
	unsigned int underflow[NF_INET_NUMHOOKS];

	/* 计数器的数量，必须等于当前表中规则的数量 */
	unsigned int num_counters;
	
	/* 旧条目的计数器 */
	struct xt_counters __user *counters;

	/* 新规则的数组。这是一个灵活数组成员，可以动态调整大小. */
	struct ipt_entry entries[0];
};

struct xt_table_info {
	/* 每个表的大小，即一个表项占用内存的大小 */
	unsigned int size;
	/* 当前表中规则数（包括初始化时的规则数） */
	unsigned int number;
	/* 该表初始化时的规则数 */
	unsigned int initial_entries;

	/* 各hook点中挂载表的首尾规则的便宜 */
	unsigned int hook_entry[NF_INET_NUMHOOKS];
	unsigned int underflow[NF_INET_NUMHOOKS];

	/* 用户自定义链 */
	unsigned int stacksize;          // 自定义链的个数
	unsigned int __percpu *stackptr; // 表中用户链的指针
	void ***jumpstack;               // 跳转栈

	/* 指向规则项的指针 */
	unsigned char entries[0] __aligned(8);
};


```
3.conntrack
