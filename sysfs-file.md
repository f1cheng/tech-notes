- how sysfile created  
  file/dir hierarchy is created via inode relative via kernfs_new_node().  
```
dir create 
error = create_dir(kobj);----------------------------------------------

how device file created?
struct kernfs_node *__kernfs_create_file(struct kernfs_node *parent,
					 const char *name,
					 umode_t mode, kuid_t uid, kgid_t gid,
					 loff_t size,
					 const struct kernfs_ops *ops,
					 void *priv, const void *ns,
					 struct lock_class_key *key)
{
	struct kernfs_node *kn;
	unsigned flags;
	int rc;

	flags = KERNFS_FILE;

	kn = kernfs_new_node(parent, name, (mode & S_IALLUGO) | S_IFREG,
			     uid, gid, flags);

https://mirrors.edge.kernel.org/pub/linux/kernel/people/mochel/doc/papers/ols-2005/mochel.pdf
https://embetronicx.com/tutorials/linux/device-drivers/sysfs-in-linux-kernel/
https://gist.github.com/strezh/b01fcd50875c214e510a81c6aa6d2a2a

/dev/gpio_drv
https://gist.github.com/strezh/b01fcd50875c214e510a81c6aa6d2a2a

device_create — creates a device and registers it with sysfs
->device_create_groups_vargs
-->device_add
--->error = device_create_file(dev, &dev_attr_uevent);
---->sysfs_create_file(&dev->kobj, &attr->attr);

error = sysfs_create_file(example_kobject, &foo_attribute.attr);
if (error) {
 	pr_debug("failed to create the foo file in /sys/kernel/kobject_example \n");
} 

https://www.cs.swarthmore.edu/~newhall/sysfstutorial.pdf


struct kobject * kobject_create_and_add ( const char * name, struct kobject * parent);

If you pass kernel_kobj to the second argument,
 it will create the directory under /sys/kernel/. 
If you pass firmware_kobj to the second argument, 
it will create the directory under /sys/firmware/. 
If you pass fs_kobj to the second argument, 
it will create the directory under /sys/fs/. 
If you pass NULL to the second argument, 
it will create the directory under /sys/.
/*Creating a directory in /sys/kernel/ */
kobj_ref = kobject_create_and_add("etx_sysfs",kernel_kobj); //sys/kernel/etx_sysfs

//This Function will be called from Init function
/*Creating a directory in /sys/kernel/ */
kobj_ref = kobject_create_and_add("etx_sysfs",kernel_kobj);
 
/*Creating sysfs file for etx_value*/
if(sysfs_create_file(kobj_ref,&etx_attr.attr)){---etx_attr!!!!!!!!!!!!
    printk(KERN_INFO"Cannot create sysfs file......\n");
    goto r_sysfs;

struct kobj_attribute etx_attr = __ATTR(etx_value, 0660, sysfs_show, sysfs_store);

}

	kn = __kernfs_create_file(parent, attr->name, mode & 0777, uid, gid,
				  PAGE_SIZE, ops, (void *)attr, ns, key);


	kn = kernfs_new_node(parent, name, (mode & S_IALLUGO) | S_IFREG,
			     uid, gid, flags);

attrs = kernfs_iattrs(kn);

simple_xattrs_init(&kn->iattr->xattrs);


In order to access this device from user space, 
you should create a device node in /dev. 
You do this by creating a virtual device 
class using class_create, then creating a device and 
registering it with sysfs using the device_create function. 
device_create will create a device file in /dev.

With this information, we can use the mknod command to make our very own device node:

# mknod foobar c 1 5



  // create device node /dev/mychardev-x where "x" is "i", equal to the Minor number
        device_create(mychardev_class, NULL, MKDEV(dev_major, i), NULL, "mychardev-%d", i);


https://olegkutkov.me/2018/03/14/simple-linux-character-device-driver/

status = sysfs_create_file(&device->dev.kobj, &fan->fine_grain_control.attr);


        // create device node /dev/mychardev-x where "x" is "i", equal to the Minor number
        device_create(mychardev_class, NULL, MKDEV(dev_major, i), NULL, "mychardev-%d", i);
    }

int device_create_file(struct device *dev,
		       const struct device_attribute *attr)
{
	int error = 0;

	if (dev) {
		WARN(((attr->attr.mode & S_IWUGO) && !attr->store),
			"Attribute %s: write permission without 'store'\n",
			attr->attr.name);
		WARN(((attr->attr.mode & S_IRUGO) && !attr->show),
			"Attribute %s: read permission without 'show'\n",
			attr->attr.name);
		error = sysfs_create_file(&dev->kobj, &attr->attr);
	}

	return error;
}



https://www.science.smith.edu/~nhowe/262/oldlabs/module.html

/* handlers for the file operations */
struct file_operations mydev_fops = {
  open   : mydev_open,            /* handler for the open() operation    */
  release: mydev_close,           /* handler for the close() operation   */
    /* NULL (default actions) */
};

/* this function is called when the module is loaded */
int mydev_init(void)
{
  ...
  devid = devfs_register( ...  
                          "mydev",     /* create /dev/mydev entry */
                          ...
                          &mydev_fops, /* file ops handlers (see above) */
                          ... );
There are several file operations that a module can i


https://www.science.smith.edu/~nhowe/262/oldlabs/src/skeldev.c


/* Check  if the Device File System (experimental) is installed */
#ifndef CONFIG_DEVFS_FS
# error "This module requires the Device File System (devfs)"
#endif

/* MODULE CONFIGURATION */

MODULE_AUTHOR("Emanuele Altieri");
MODULE_DESCRIPTION("Basic Device Driver");
MODULE_LICENSE("GPL");

static devfs_handle_t skel_handle;    /* handle for the device     */
static char* skel_name = "skeldev";   /* create /dev/skeldev entry */

static struct file_operations skel_fops = { /* NULL (default actions) */ };

/* Module Initialization */
static int __init skel_init(void)
{
	/* register the module */
	SET_MODULE_OWNER(&skel_fops);
	skel_handle = devfs_register
		(
		 NULL,                      /* parent dir in /dev (none)     */
		 skel_name,                 /* /dev entry name (skeldev)     */
		 DEVFS_FL_AUTO_DEVNUM |     /* automatic major/minor numbers */
		 DEVFS_FL_AUTO_OWNER,
		 0, 0,                      /* major/minor (not used)        */
		 S_IFCHR,                   /* character device              */
		 &skel_fops,                /* file ops handlers             */
		 NULL                       /* other                         */
		 );
	if (skel_handle <= 0)
		return(-EBUSY);

	return(0);
} /* skel_init() */

/* Module deconstructor */
static void __exit skel_exit(void)
{
	devfs_unregister(skel_handle);
} /* skel_exit() */

/* Specify init and exit functions */

=====

associate as below for dummy file with operation.

https://github.com/razvanvirtan/unikraft/blob/105c61821f5d8cfefa231732297668be92ae7c23/lib/ukswrand/dev.c#L90

dev added into list or array devfs[xx].

=======
int devfs_register(struct devfs_file *devf)
{
    devfs_files[devfs_nexti++] = devf;
    return 0;
}

or=====
	memcpy(dev->name, name, len + 1);
	dev->driver = drv;
	dev->flags = flags;
	dev->active = 1;
	dev->refcnt = 1;

	/* Insert device into device list. This must be done while holding
	 * the devfs lock. We optimize for success and reduce lock holder time
	 * by doing the check for name collision only here, instead of keeping
	 * the lock held the whole time.
	 */
	uk_mutex_lock(&devfs_lock);

	if (unlikely(device_lookup(name) != NULL)) {
		uk_mutex_unlock(&devfs_lock);
		free(dev);

		return EEXIST;
	}

	dev->next   = device_list;
	device_list = dev;

====================
static struct devops random_devops = {
	.open = dev_noop_open,
	.close = dev_noop_close,
	.read = dev_random_read,
	.write = dev_random_write,
	.ioctl = dev_noop_ioctl,
};

static struct driver drv_random = {
	.devops = &random_devops,
	.devsz = 0,
	.name = DEV_RANDOM_NAME
};

static struct driver drv_urandom = {
	.devops = &random_devops,
	.devsz = 0,
	.name = DEV_URANDOM_NAME
};

static int devfs_register(struct uk_init_ctx *ictx __unused)
{
	int rc;

	uk_pr_info("Register '%s' and '%s' to devfs\n",
		   DEV_URANDOM_NAME, DEV_RANDOM_NAME);

	/* register /dev/urandom */
	rc = device_create(&drv_urandom, DEV_URANDOM_NAME, D_CHR, NULL);
	if (unlikely(rc)) {
		uk_pr_err("Failed to register '%s' to devfs: %d\n",
			  DEV_URANDOM_NAME, rc);
		return -rc;
	}

	/* register /dev/random */
	rc = device_create(&drv_random, DEV_RANDOM_NAME, D_CHR, NULL);
	if (unlikely(rc)) {
		uk_pr_err("Failed to register '%s' to devfs: %d\n",
			  DEV_RANDOM_NAME, rc);
		return -rc;
	}

	return 0;
}

devfs_initcall(devfs_register);





====


例如，如果我们要注册的设备驱动程序叫做DEVICE_NAME，其主设备号为MAJOR_NR，次设备号为MINOR_NR，缺省的文件操作为 device_fops：则该设备驱动程序的init_module（）函数和cleanup_module（）函数如下：
int init_module(void)
{
int ret;
if ((ret = register_chrdev(MAJOR_NR, DEVICE_NAME, &device_fops)) == 0)
return ret;
}
void cleanup_module(void)
{
unregister_chrdev(MAJOR_NR, DEVICE_NAME);
}

对以上代码进行改写以支持设备文件系统（假定设备入口点的名字为DEVICE_ENTRY)
#include <linux/devfs_fs_kernel.h>
devfs_handle_t devfs_handle;
int init_module(void)
{
int ret;
if ((ret = devfs_register_chrdev(MAJOR_NR, DEVICE_NAME, &device_fops)) == 0)
return ret;
devfs_handle = devfs_register(NULL, DEVICE_ENTRY, DEVFS_FL_DEFAULT,
MAJOR_NR, MINOR_NR, S_IFCHR | S_IRUGO | S_IWUSR,
&device_fops, NULL);
}
void cleanup_module(void)
{
devfs_unregister_chrdev(MAJOR_NR, DEVICE_NAME);
devfs_unregister(devfs_handle);
}

（2）在Devfs名字空间中创建一个目录

devfs_mk_dir()用来创建一个目录，这个函数返回Devfs的句柄，这个句柄用作devfs_register的参数dir。 例如，为了在“/dev/mydevice”目录下创建一个设备设备入口点，则进行如下操作：
devfs_handle = devfs_mk_dir(NULL, "mydevice", NULL);
devfs_register(devfs_handle, DEVICE_ENTRY, DEVFS_FL_DEFAULT,
MAJOR_NR, MINOR_NR, S_IFCHR | S_IRUGO | S_IWUSR,
&device_fops, NULL);

（3）注册一系列设备入口点

如果一个设备有几个次设备号，就说明同一个设备驱动程序控制了几个不同的设备，例如主IDE硬盘的主设备号为3，但其每个分区都有一个从设备号，例如 /dev/had2的从设备号为2。在Devfs下，每个从次设备号也有一个目录，例如/dev/ide0/,/dev/ide1/等，也就是说，每个次 设备号都有一个设备入口点，于是就可以调用devfs_register_series来创建一系列的设备入口点。设备入口点的名字以printf()函 数中format参数的形式来创建。

注册DEVICE_NR设备入口点（次设备号从MINOR_START开始）的操作如下：
devfs_handle = devfs_mk_dir(NULL, "mydevice", NULL);
devfs_register_series(devfs_handle, "device%u", max_device, DEVFS_FL_DEFAULT,
MAJOR_NR, MINOR_START, S_IFCHR | S_IRUGO | S_IWUSR,
&device_fops, NULL);

（4）块设备

注册和注销块设的函数为：
devfs_register_blkdev（）
devfs_unregister_blkdev （）



===

这个写在初始化函数中，register_chrdev之后

#ifdef CONFIG_DEVFS_FS

   devfs_gpf_dir = devfs_mk_dir(NULL, "gpf", NULL);

   devfs_gpf_raw = devfs_register(devfs_gpf_dir, "key",   DEVFS_FL_DEFAULT, gpf_Major, 0, S_IFCHR | S_IRUSR | S_IWUSR, &s3c2410_fops, NULL);

#endif

分析：devfs_gpf_dir保存创建的设备目录 ”gpf”，devfs_register第一个参数是devfs_gpf_dir,第二个参数设备名 ”key”，第四个参数是主设备号，第七个参数是file_operations的结构体。

这样就会在/dev下建立一个/gpf/key的设备文件，并通过主设备号指向刚注册的字符设备，相当于手动创建 mknod 文件路径c 主设备号 从设备号


int __init init_netlink(void)
{
	if (devfs_register_chrdev(NETLINK_MAJOR,"netlink", &netlink_fops)) {
		printk(KERN_ERR "netlink: unable to get major %d\n", NETLINK_MAJOR);
		return -EIO;
	}
	devfs_handle = devfs_mk_dir (NULL, "netlink", 7, NULL);
	/*  Someone tell me the official names for the uppercase ones  */
	make_devfs_entries ("route", 0);
	make_devfs_entries ("skip", 1);
	make_devfs_entries ("USERSOCK", 2);
	make_devfs_entries ("fwmonitor", 3);
	make_devfs_entries ("ARPD", 8);
	make_devfs_entries ("ROUTE6", 11);
	make_devfs_entries ("IP6_FW", 13);
	devfs_register_series (devfs_handle, "tap%u", 16, DEVFS_FL_DEFAULT,
			       NETLINK_MAJOR, 16,
			       S_IFCHR | S_IRUSR | S_IWUSR,
			       &netlink_fops, NULL);
	return 0;
}

int devfs_register_tape(const char *name)
{
	char tname[32], dest[64];
	static unsigned int tape_counter;
	unsigned int n = tape_counter++;

	sprintf(dest, "../%s", name);
	sprintf(tname, "tapes/tape%u", n);
	devfs_mk_symlink(tname, dest);

	return n;
}

!!!!!!!!!
static inline int register_chrdev(unsigned int major, const char *name,
				  const struct file_operations *fops)
{
	return __register_chrdev(major, 0, 256, name, fops);
}

		/*
		 * Create /dev/port?
		 */
		if ((minor == DEVPORT_MINOR) && !arch_has_dev_port())
			continue;

		device_create(&mem_class, NULL, MKDEV(MEM_MAJOR, minor),
			      NULL, devlist[minor].name);

int device_create_file(struct device *dev,
		       const struct device_attribute *attr)
{
	int error = 0;

	if (dev) {
		WARN(((attr->attr.mode & S_IWUGO) && !attr->store),
			"Attribute %s: write permission without 'store'\n",
			attr->attr.name);
		WARN(((attr->attr.mode & S_IRUGO) && !attr->show),
			"Attribute %s: read permission without 'show'\n",
			attr->attr.name);
		error = sysfs_create_file(&dev->kobj, &attr->attr);
	}

	return error;
}


  ```
