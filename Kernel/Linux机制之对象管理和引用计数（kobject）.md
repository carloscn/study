# Linux机制之对象管理和引用计数（kobject/ktype/kset）
内核很多地方都要跟踪记录C语言中结构体的实例。尽管这些对象用法不太一样，但各个子系统的操作都非常类似，例如引用计数。这导致了代码的复制。由于这个是个糟糕的问题，因此内核采用了一种通用的方法来管理这个行为和内核对象。引入这个框架的目的是为了防止代码复制，同时也为内核不同部分管理的对象提供了一些视图，在内核的许多部分可以有效的使用相关信息，例如电源，例如驱动。一般性内核对象机制可以用于执行下列对象操作：
* 引用计数
* 管理对象链表（集合）
* 集合加锁
* 将对象属性导出到用户空间（通过sysfs文件系统）

本文整理范本[^1][^2]

# 1. kobject
本文会围绕kobject、ktype和kset三个概念进行介绍，我们先大概了解一下相关概念以及它们之间的关系：

**kobject**在内核中应用最多的就是设备驱动模型————总线、设备、驱动、类的管理都使用了kobject，但是kobject并不只为设备驱动模型服务，它是内核中的通用对象模型，用来为内核中各部分的对象管理提供统一视图，其实现在内核的lib/目录下。

 **kobject ，它也不过是内核里的一个结构体而已**：

```C
struct kobject {
    const char        *name;
    struct list_head    entry;
    struct kobject        *parent;
    struct kset        *kset;
    struct kobj_type    *ktype;
    ......// 省略一些暂时不想看到的东西
};
```
每一个 kobject 对应 文件系统 /sys 里的一个 目录，目录的名字就是结构体中的 name：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220915095232.png)

kobject一般都不会单独使用，这样是没有意义的，它总是内嵌到其他结构体中，例如字符设备的定义：
```C
struct cdev {
    struct kobject kobj;
    struct module *owner;
    const struct file_operations *ops;
    struct list_head list;
    dev_t dev;
    unsigned int count;
};
```

我们说kobject只是通用对象的表示，其自身不包含任何特定于某个模块的属性，因此我们更关心的是kobject的外围结构。每个外围结构体所在的模块都应该定义一个ktype，这个ktype就完成了由通用向特殊的过渡。因此kobject可看做其他所有外围结构的基类，它抽象出一些通用的属性（如引用计数）以及通用接口，而每个外围结构各自完成这些接口的内部实现，这里使用内嵌的方式来体现这种继承关系。

使用内嵌而不用指针也是为了方便通过kobject来反向定位其外围结构的实例，通过我们熟知的container_of()宏便可以做到。由于kobject是系统统一管理的，因此先找到kobject对象进而跟踪到其代表的具体对象是很常见的做法。

kobject在内核中不会单独出现，并且散落在内核的各个角落，因此很难追踪到每一个kobject对象。好在 **Linux内核将所有的kobjects与虚拟文件系统sysfs紧密结合起来**，这样就让所有kobjects可视化和层次化。kobject描述了sysfs虚拟文件系统中的层级结构，一个kobject对象就对应了sysfs中的一个目录，而sysfs中的目录结构也体现在各个kobjects之间的父子关系。

kobject由struct kobject定义：
```C
#include <linux/kobject.h>
struct kobject {
    const char      *name; /* kobject对象的名字，对应sysfs中的目录名 */
    struct list_head    entry; /* 在kset中的链表节点 */
    struct kobject      *parent; /* 用于构建sysfs中kobjects的层次结构，指向父目录 */
    struct kset     *kset; /* 所属kset */
    struct kobj_type    *ktype; /* 特定对象类型相关，用于跟踪object及其属性 */
    struct sysfs_dirent *sd; /* 指向该目录的dentry私有数据 */
    struct kref     kref; /* kobject的引用计数，初始值为1 */
    unsigned int state_initialized:1; /* kobject是否初始化，由kobject_init()设置 */
    unsigned int state_in_sysfs:1; /* 是否已添加到sysfs层次结构中 */
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1; /* 是否忽略uevent事件 */
};
```

新建一个kobject对象需要两步：初始化struct kobject结构，添加到sysfs目录结构中。  

* **初始化一个kobject对象**：

	`void kobject_init(struct kobject *kobj, struct kobj_type *ktype);`
	
	这个函数需要传入两个参数，kobj和ktype必须不为NULL，由于kobj都是嵌入到其他结构体，所以一般传kobj参数的方式形如&pcdev->kobj。
	* 该函数即完成对kobj的初始化：
	* 初始化kobj->kref引用计数为初始值1；
	* 初始化kobj->entry空链表头；
	* kobj->ktype = ktype;
	* 然后将kobj->state_initialized置为1，表示该kobject已初始化。

* **初始化之后，通过kobject_add()将kobj添加到系统中：**
	
	`int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...);`
	
	这个函数给kobj指定一个名字，这个名字也就是其在sysfs中的目录名，parent用来指明kobj的父节点，即指定了kobj的目录在sysfs中创建的位置。如果这个kobj要加入到一个特定的kset中，则在kobject_add()必须给kobj->kset赋值，此时parent可以设置为NULL，这样kobj会自动将kobj->kset对应的对象作为自己的parent。如果parent设置为NULL，且没有加入到一个kset中，kobject会被创建到/sys顶层目录下。

* **设置对象的名字是通过下面的接口完成的**：
	
	`int kobject_set_name(struct kobject *kobj, const char *fmt, ...);`
	
	这个函数给kobj的name成员赋值。注意这里面是通过kmalloc给name分配内存的。相应的获取一个kobject对象的名字的接口为：
	
	`const char *kobject_name(const struct kobject * kobj);`
	
	如果在添加kobj之后要给kobj改名字，则使用kobject_rename()接口，而不要直接去操作kobj->name，因为涉及到内存重新分配以及sysfs中目录的改变。
	
	也可以通过下面的接口来一次性完成kobject_init()和kobject_add()过程：
	
	`int kobject_init_and_add(struct kobject *kobj,struct kobj_type *ktype, struct kobject *parent, const char *fmt, ...);`

	参数含义和上述接口相同。

# 2 kset
我们前面说了，每一个 kobj 对应 文件系统/sys里的一个 目录，那么每一个kset 都包含了一个 kobj，那样的话，kset也对应于/sys里的一个目录简单来说，kset与kobj 都是目录，既然是目录，那么在就是一个树状结构，每一个目录都将有一个 父节点:
* 在kset中使用kset.kobj->parent 指定
* 在kboject中使用kobj->parent 指定

显然，整个树状目录结构，都是通过kobj来构建的，只不过有些kobj嵌在kset里，分析目录结构时把kset当成一个普通的kobj会好理解很多。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220915100233.png)

那么kset **有何特殊之处呢**？

我们可以看到 kset 结构中有一个链表，它目录下的 一些相同类型的kobj会在创建时链入链表，也就是说Kset是一些kobj的集合,光说还是比较抽象的，那么我们就来看看 /sys 目录底下的这些东西，哪些是kset？ 哪些是kobj 结构又是怎样的。

看过代码的应该知道，想要在/sys 目录下创建目录，那就要构造 kset 或者 kobj ，设置并注册。

对于kobject设置注册方法是：

```text
kobject_create_and_add
kobject_init_and_add
```

对于kset设置注册方法是：

```text
kset_create_and_add
```

我在这3个函数中增加了prink打印语句，打印内核创建的每一个 kobj 或者 kset 的名字，以及父节点的名字，甚至它指向的kset的kobj的名字。

原本我以为，较高层次的目录会是kset，因为它是个集合嘛，然而并不全是。打印信息如下：

```text
the kset name is devices,no parent
    the kset name is system,parent's name is devices
    the kobject name is virtual,parent'
the kset name is bus,no parent
the kset name is class,no parent
the kset name is module,no parent
the kobject name is fs,no parent
the kobject name is dev,no parent
    the kobject name is block,parent's name is dev
    the kobject name is char,parent'
the kobject name is firmware,no parent
the kobject name is kernel,no parent
    the kset name is slab,parent's name is kernel
```

写着no parent的，在/sys/目录下可以找到它们，对于devices、bus、class、module它们是kset；对于fs、dev、firmware、kernel、block 它们是kobj。

而且，我们还可以发现：

1、 kset 底下可以放 kset （但是无法链入链表，分析代码时会知道）

```text
the kset name is devices,no parent
    the kset name is system,parent's name is devices
```

2、 kset 底下可以放 kobj （可以链入链表，也可以不链入）

```text
the kset name is devices,no parent
    the kobject name is virtual,parent's name is devices
```

3、 kobject 底下可以放 kset （显然没链表的概念了）

```text
the kobject name is kernel,no parent
   the kset name is slab,parent's name is kernel
```

4、 kobj 底下放 kobj （同样没有链表的概念）

```text
the kobject name is dev,no parent
   the kobject name is block,parent's name is dev
```

如下图：黄色代表Kset，蓝色代表Kobject

![](https://raw.githubusercontent.com/carloscn/images/main/typorasys1.svg)

至此我们对kset kobj它们之间的联系应该有一个比较浅显的认识了。下面分析代码进一步摸索 , 先把图贴上来，虚线表示可能的其它一些连接情况。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220915102106.png)


先看kobject_create_and_add的实现：

```c
struct kobject*kobject_create_and_add(const char *name, struct kobject *parent)
{
     struct kobject *kobj;
     kobj = kobject_create();
     retval = kobject_add(kobj, parent, "%s", name);
     return kobj;
}
```

再看它的调用过程：

```C
kobject_create_and_add
    kobject_create        // kobj =kzalloc(sizeof(*kobj), GFP_KERNEL);
    kobject_init              //kobj->ktype = ktype;
    kobject_init_internal    // kref_init(&kobj->kref);
    kobject_add
    kobject_add_varg    // retval = kobject_set_name_vargs(kobj,fmt, vargs); 
					        // kobj->parent = parent;
    kobject_add_internal
	if (kobj->kset) {    // kobj->kset == NULL 不执行
		if (!parent)
			parent = kobject_get(&kobj->kset->kobj);
            kobj_kset_join(kobj);
            kobj->parent = parent;
        }
        create_dir
```

kobject_create_and_add 函数从构建一个kobject到在sysfs中创建路径一气呵成，其中没有关于kset的设置，仅仅是设置了parent ktype

如果 kobject_create_and_add 传入的 parent 参数是一个普通的kobject ，那么就与应于图中的③与⑤的关系

如果 kobject_create_and_add 传入的 parent 参数是一个kset->kobject，那么就与应于图中的③与④的关系

```text
priv->kobj.kset =bus->p->drivers_kset;  // 设置它所属的Kset
           error = kobject_init_and_add(&priv->kobj, &driver_ktype,NULL, "%s", drv->name);
```

再看kobject_init_and_add的实现以及调用：

```C
int kobject_init_and_add(struct kobject *kobj, struct kobj_type *ktype,struct kobject *parent, const char *fmt, ...) {
    kobject_init(kobj, ktype);
    retval = kobject_add_varg(kobj, parent, fmt, args);
}

kobject_init_and_add   
    kobject_init    // kobj->ktype = ktype;
	kobject_init_internal    // kref_init(&kobj->kref);
    kobject_add_varg    // retval = kobject_set_name_vargs(kobj,fmt, vargs); 
				    // kobj->parent = parent;
    kobject_add_internal
	if (kobj->kset) {    // 将kobject添加到它所指向的kset->list中
		if (!parent)
			parent = kobject_get(&kobj->kset->kobj);
            kobj_kset_join(kobj);
            kobj->parent = parent;  // 如果没有parent 将它所属的kset->kobj作为它的parent
        }
        create_dir
```

与kobject_create_and_add 相比，就是少了构建kobkject，然而这样给了我们设置kset的机会，同时往往不会设置parent，对应于图中的①与④的关系

再看kset_create_and_add的实现以及调用：

```C
struct kset*kset_create_and_add(const char *name,
struct kset_uevent_ops *uevent_ops,
struct kobject *parent_kobj)
     {
 ...
        kset= kset_create(name, uevent_ops, parent_kobj);
 ...
        error= kset_register(kset);
      }
kset_create_and_add
       kset_create
           kset = kzalloc(sizeof(*kset), GFP_KERNEL);
           retval = kobject_set_name(&kset->kobj, name);
           kset->uevent_ops = uevent_ops;       
           kset->kobj.parent = parent_kobj;
           kset->kobj.ktype = &kset_ktype;
           kset->kobj.kset = NULL;
       kset_register
           kset_init(k);
               err = kobject_add_internal(&k->kobj);
parent = kobject_get(kobj->parent);
if (kobj->kset) {    // kobj->kset==NULL 不执行
                      ....
                  }
                   error = create_dir(kobj);
```

kset_create_and_add 无法将创建kset->kobj.kset 指向任何kset

但是kset->kobj.parent 还是能和kobj.parent指向普通的kobj 或者包含在kset里的kobj。

如果 kset_create_and_add 传入的 parent 参数 是一个普通的kobject ，那么就对应于图中的④与⑤的关系

如果 kset_create_and_add 传入的 parent 参数 是一个kset->kobject，那么就对应于图中的②与④的关系

还有一种情况就是，创建一个 kset 并设置kset.kobject.kset

然后调用 kset_register

kset_register的调用过程：

```C
kset_register
    kset_init(k);
        err = kobject_add_internal(&k->kobj);
		parent = kobject_get(kobj->parent);
		if(kobj->kset) {    // 将kobject 添加到它所指向的kset->list中
			if (!parent)
				parent = kobject_get(&kobj->kset->kobj);
            kobj_kset_join(kobj);
            kobj->parent = parent;  // 如果没有parent 将它所属的kset->kobj作为它的parent
        }
        error = create_dir(kobj);
```

对应于图中④与⑥的关系。

上面代码的细节，比如如何在/sys/创建路径请参考：

[http://blog.csdn.net/lizuobin2/article/details/51511336](https://link.zhihu.com/?target=http%3A//blog.csdn.net/lizuobin2/article/details/51511336)

到此，我们应该对 kobject kset sysfs 之间的目录关系比较清楚了，但是我们至少还应该看看ktype。

kobject 包括了kset：

```c
struct kobject {
	const char        *name;
	struct list_head    entry;
	struct kobject        *parent;
	struct kset        *kset;
	struct kobj_type    *ktype;
	struct kref        kref;
 
    ……
    };
```

ktype 由一个release函数、一个sysfs_ops、一个指针数组（二维指针）组成：

```c
struct kobj_type {
    void(*release)(struct kobject *kobj);
    struct sysfs_ops *sysfs_ops;
    struct attribute **default_attrs;  
    //struct attribute *default_attrs[]
 };
```

1、release 函数

每一个Kobject 都必须有一个release方法，有意思的是，release 函数并没有包括在kobject自身内，而是包含在它的结构体成员Ktype内。而且kobject在调用release之前应该保持稳定（不明白抄自LDD3）。

2、

```c
struct attribute **default_attrs
 
struct attribute {
	const char        *name;
	struct module        *owner;
	mode_t           mode;
 };
```

default_attrs 指向的地方是个指针数组，这些指针的类型为attribute ，那么这些attribute就是该kobject的属性了，name是属性的名字，在kobject目录下表现为文件 ，owner 指向模块的指针（如果有的话），那么该模块负责实现这些属性。mode 是保护位，通常是S_IRUGO，可写的则用S_IWUSR 仅为root提供写权限。default_attrs最后一个元素必须为0，要不然它找不着北~

3、sysfs_opes 实现属性的方法

```c
struct sysfs{
      ssize_t *show(struct kobject *kobject, struct attribute*attr,char *buf);
      ssize_t *store(struct kobject *kobject, struct attribute*attr,char *buf, size_t size);
 
}
```

在内核里，一类设备往往使用相同的show ， store函数。

附上一个例子：在linux2.6.32.2下编译通过

```c
#include<linux/device.h>  
#include <linux/module.h>  
#include <linux/kernel.h>  
#include <linux/init.h>  
#include <linux/string.h>  
#include <linux/sysfs.h>  
#include <linux/stat.h>  
   
MODULE_LICENSE("Dual BSD/GPL"); 
 
---------------------------------------default_attrs---------------------------
/*对应于kobject的目录下的一个文件,Name成员就是文件名*/
struct attribute test_attr = {
   .name = "kobj_config",  
   .mode = S_IRWXUGO,  
};
 
static struct attribute *def_attrs[] ={
   &test_attr,  
   NULL,  
};  
---------------------------------------sysfs_ops------------------------------- 
/*当读文件时执行的操作*/
ssize_t kobj_test_show(struct kobject*kobject, struct attribute *attr,char *buf) 
{  
   printk("have show.\n");  
   printk("attrname:%s.\n", attr->name);  
   sprintf(buf,"%s\n",attr->name);  
   return strlen(attr->name)+2;  
} 
 
/*当写文件时执行的操作*/
ssize_t kobj_test_store(struct kobject*kobject,struct attribute *attr,const char *buf, size_t count) 
{  
   printk("havestore\n");  
   printk("write: %s\n",buf);  
   return count;  
}  

//kobject对象的操作  
struct sysfs_ops obj_test_sysops =   
{
   .show = kobj_test_show,  
   .store = kobj_test_store,  
};
---------------------------------------release------------------------------------------------------- 
/*release方法释放该kobject对象*/
void obj_test_release(struct kobject*kobject)
{  
   printk("eric_test: release .\n");  
}
---------------------------------------kobj_type-----------------------------------------------------          
/*定义kobject对象的一些属性及对应的操作*/
struct kobj_type ktype =   
{
   .release = obj_test_release,  
   .sysfs_ops=&obj_test_sysops,  
   .default_attrs=def_attrs,  
}; 
     
struct kobject kobj;//声明kobject对象 
  
static int kobj_test_init(void)
{  
   printk("kboject test init.\n");  
   kobject_init_and_add(&kobj,&ktype,NULL,"kobject_test");/
  return 0;  
}  
   
static void kobj_test_exit(void)
{  
   printk("kobject test exit.\n");  
   kobject_del(&kobj);  
}  
   
module_init(kobj_test_init); 
module_exit(kobj_test_exit);
```

执行程序，可以看到在sys目录多了kobject test目录，里面有kobj_config文件并且可读。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220915101051.png)

# ktype
ktype是由struct kobj_type来定义的：
```C
#include <linux/kobject.h>
struct kobj_type {
void (*release)(struct kobject *kobj);
const struct sysfs_ops *sysfs_ops;
struct attribute **default_attrs;
const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
const void *(*namespace)(struct kobject *kobj);
};
```

kobj_type都必须实现release方法，该方法由kobject的外层结构来定义，用来释放特定模块相关的kobject资源。

default_attrs定义了一系列默认属性，default_attrs是一个二级指针，可以对每个kobject设置多个默认属性（最后一个属性用NULL填充）。

```C
struct attribute {
    const char      *name; /* 属性名字 */
    umode_t         mode; /* 用户访问模式，在<linux/stat.h>中定义 */
};
```

例如：
```C
static struct attribute *i2c_adapter_attrs[] = {
    &dev_attr_name.attr,
    &dev_attr_new_device.attr,
    &dev_attr_delete_device.attr,
    NULL
};
```

可见，kobj_type是由具体模块定义的，每一个属性都对应着kobject目录下的一个文件，这样可以在用户态通过读写属性文件，来完成对该属性值的读取和更改。

而sysfs_ops定义了属性的操作：

```C
struct sysfs_ops {
    ssize_t (*show)(struct kobject *, struct attribute *,char *);
    ssize_t (*store)(struct kobject *,struct attribute *,const char *, size_t);
    const void *(*namespace)(struct kobject *, const struct attribute *);
};
```

也就是说，default_attrs数组中所有属性的操作都是由sysfs_ops来完成的。
在kobject的容器（外围结构）中，一般会定义xxx_add_attrs()和xxx_remove_attrs()提供增加和删除额外属性的接口。例如对于bus_ktype，默认属性是空的，因此我们看到/sys/bus/目录下是没有属性文件的，而每个子目录的属性可能不同，因此都需要动态地去定义属性列表并据此创建属性文件。动态属性一般通过__ATTR()宏来定义。

创建属性文件最终都是通过sysfs_create_file()或sysfs_create_group()来完成的。其中sysfs_create_group(kobj, grp)用来创建一组属性文件，需要定义struct attribute_group结构的属性集，这里需要说明的是，其中name成员如果为NULL，则直接在kobj目录下创建各个属性文件，如果name不为NULL，则会创建一个名为name的目录，然后在该目录下创建各个属性文件。

```C
struct attribute_group {
    const char      *name;
    umode_t         (*is_visible)(struct kobject *,
                          struct attribute *, int);
    struct attribute    **attrs;
};
```

## 引用计数kref
kref成员是object对象的引用计数，初始值为1，通过kref_get()和kref_put()可对该计数进行增减操作。kref_get()和kref_put()是内核中通用的引用计数的操作，针对kobject，使用下面两个封装函数：
```C
struct kobject *kobject_get(struct kobject *kobj);
void kobject_put(struct kobject *kobj);
```

为方便跟踪一个kobject，在使用kobject之前，都要使用kobject_get()将参数kobj的kref加1，该函数返回kobj自身的指针。kobject_put()则将kref减1，当kref减到0时，则kobject_release()被调用来释放kobject资源。

kobject_release()会调用ktype的release方法做实际的释放操作，并释放为name分配的内存。如果有必要，还会向用户态发送KOBJ_REMOVE事件，并通过kobject_del()将kobject从sysfs中删除。

## 与sysfs关联 —— sysfs_dirent

读写sysfs中的文件（即每个attribute）的操作函数集为sysfs_file_operations：

```C
const struct file_operations sysfs_file_operations = {
    .read       = sysfs_read_file,
    .write      = sysfs_write_file,
    .llseek     = generic_file_llseek,
    .open       = sysfs_open_file,
    .release    = sysfs_release,
    .poll       = sysfs_poll,
};
```

在读写sysfs的文件时，是通过文件的dentry找到和sysfs关联的struct sysfs_dirent结构，sysfs中每一个目录或文件都有其对应的一个struct sysfs_dirent结构的实例，而sysfs_dirent的s_parent成员用于构建sysfs中的层次结构。
那么，对于一个属性文件的sysfs_dirent结构，可通过其s_parent找到其所在目录的kobject实例，进而找到文件操作对应的kobj->ktype->sysfs_ops方法集合。相应的，kobject的sd成员即指向该目录对应的sysfs_dirent实例。

在读写一个sysfs里的属性文件时，其中read操作会去调用其所在目录对应的kobject的kobj->ktype->sysfs_ops的show方法完成属性读取，write操作会去调用kobj->ktype->sysfs_ops的store方法完成属性写入并使其生效。open/release方法用于打开/关闭文件。poll用来探测文件内容是否改变（如果相应kobject实现了poll/select的事件通知），poll方法返回POLLERR|POLLPRI事件，如果发现文件内容有变则需要重新打开文件或seek到文件头并重新读取才能读到新的内容。

实际上，上述sysfs_ops的show和store方法也是简单的包裹函数，实际调用了具体模块的xxx_attribute的show和store方法（在实现一个属性的操作时，如果是很简单的属性读取和配置，可以复用struct kobj_attribute，这种属性一般只是一个整型或字符串，如果是比较复杂的模块和属性，则可自定义形如xxx_attribute的结构及其show/store方法）。

## Uevents
在一个kobject的状态变化时（新注册、注销、重命名等），都会广播出一个对应的事件通知（通常用户态会去接收并处理），通知事件的接口为：

`int kobject_uevent(struct kobject *kobj, enum kobject_action action);` 或者 `int kobject_uevent_env(struct kobject *kobj, enum kobject_action action, char *envp_ext[]);`

第二个接口可以传递额外的环境变量（“额外”的意思是说，即使envp_ext为NULL，也会传递基本的”ACTION=%s”、”DEVPATH=%s”、”SUBSYSTEM=%s”、”SEQNUM=%llu”）。action的可取值如下定义：

```C
enum kobject_action {
    KOBJ_ADD,
    KOBJ_REMOVE,
    KOBJ_CHANGE,
    KOBJ_MOVE,
    KOBJ_ONLINE,
    KOBJ_OFFLINE,
    KOBJ_MAX
};
```

这些动作在发送到用户态时是通过字符串来表达的，其对应关系为：

```C
static const char *kobject_actions[] = {
    [KOBJ_ADD] =        "add",
    [KOBJ_REMOVE] =     "remove",
    [KOBJ_CHANGE] =     "change",
    [KOBJ_MOVE] =       "move",
    [KOBJ_ONLINE] =     "online",
    [KOBJ_OFFLINE] =    "offline",
};
```


# Ref
[^1]:[Linux设备驱动之Kobject、Kset](https://blog.csdn.net/lizuobin2/article/details/51523693)
[^2]:[Linux内核中的kobject和kset介绍](https://blog.csdn.net/jasonchen_gbd/article/details/78013643)