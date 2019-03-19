### 一 SPDK简介
SPDK是intel公司为NVME ssd硬件开发的一套用于加速硬件性能的一款开发套件，支持不同层面的lib库，包括nvme ssd driver、ioat、bdev等。
###二 源码目录

 - app
	 - app/iscsi_tgt: iscsi target
	 - app/nvmf_tgt:  NVMe-oF target
	 - app/iscsi_top：iscsi top工具 类似于linux top，用来监控iscsi
	 - app/trace：iscsi target和nvme-of target  trace工具
	 - app/vhost：将virtio控制器呈现给基于qemu的虚拟机，并对IO进行处理
 - build
 - doc：spdk 下的doc文件
 - dpdk：spdk调用了dpdk的很多基础库
 - etc：各类型使用方式的基本配置
 - examples：示例代码
 - lib：开发库
 - mk：makefile文件
 - scripts：脚本及环境配置相关
 - test：各模块功能性能测试
 
 ###三 示例代码
 本文主要从hello_world代码开始，分析应用如何使用nvme ssd设备，hello_world代码利用ssd存储数据并读出。
 

```
int main(int argc, char **argv){
	int rc;
	struct spdk_env_opts opts;

	spdk_env_opts_init(&opts);
	opts.name = "hello_world";
	opts.shm_id = 0;
	if (spdk_env_init(&opts) < 0) {//前半部分代码主要都是用来初始化spdk环境及配置基本项
		fprintf(stderr, "Unable to initialize SPDK env\n");
		return 1;
	}

	printf("Initializing NVMe Controllers\n");

	rc = spdk_nvme_probe(NULL, NULL, probe_cb, attach_cb, NULL);//调用接口将机器内的nvme ssd加入列表
	if (rc != 0) {
		fprintf(stderr, "spdk_nvme_probe() failed\n");
		cleanup();
		return 1;
	}

	if (g_controllers == NULL) {
		fprintf(stderr, "no NVMe controllers found\n");
		cleanup();
		return 1;
	}

	printf("Initialization complete.\n");
	hello_world();//读写操作
	cleanup();//停止attach 对应的nvme ssd
	return 0;
}

```
主要流程：

 1. 初始化spdk环境
 2. 扫描nvme ssd并加载
 3. 读写操作
 4. 卸载nvme ssd
其中，加载nvme ssd如何做到？众多的ssd又如何管理？

```
static bool
probe_cb(void *cb_ctx, const struct spdk_nvme_transport_id *trid,
	 struct spdk_nvme_ctrlr_opts *opts)
{
	printf("Attaching to %s\n", trid->traddr);

	return true;
}
```

```

static void
attach_cb(void *cb_ctx, const struct spdk_nvme_transport_id *trid,
	  struct spdk_nvme_ctrlr *ctrlr, const struct spdk_nvme_ctrlr_opts *opts)
{
	int nsid, num_ns;
	struct ctrlr_entry *entry;
	struct spdk_nvme_ns *ns;
	const struct spdk_nvme_ctrlr_data *cdata = spdk_nvme_ctrlr_get_data(ctrlr);

	entry = malloc(sizeof(struct ctrlr_entry));
	if (entry == NULL) {
		perror("ctrlr_entry malloc");
		exit(1);
	}

	printf("Attached to %s\n", trid->traddr);

	snprintf(entry->name, sizeof(entry->name), "%-20.20s (%-20.20s)", cdata->mn, cdata->sn);

	entry->ctrlr = ctrlr;
	entry->next = g_controllers;
	g_controllers = entry;

	//namespace==scis lun，不同的ssd支持的namespace数量不一致
	num_ns = spdk_nvme_ctrlr_get_num_ns(ctrlr);
	printf("Using controller %s with %d namespaces.\n", entry->name, num_ns);
	for (nsid = 1; nsid <= num_ns; nsid++) {
		ns = spdk_nvme_ctrlr_get_ns(ctrlr, nsid);
		if (ns == NULL) {
			continue;
		}
		register_ns(ctrlr, ns);
	}
}

```
不难发现，probe_cb在扫描对应的nvme ssd设备后进行回调，attach_cb在被加载后回调。由于nvme ssd有众多的namespace，相当于机械硬盘的LUN，因此需要进行管理，这里利用了两个数据结构。

```
struct ctrlr_entry {
	struct spdk_nvme_ctrlr	*ctrlr;
	struct ctrlr_entry	*next;
	char			name[1024];
};

struct ns_entry {
	struct spdk_nvme_ctrlr	*ctrlr;
	struct spdk_nvme_ns	*ns;
	struct ns_entry		*next;
	struct spdk_nvme_qpair	*qpair;
};

static struct ctrlr_entry *g_controllers = NULL;//管理所有的nvme ssd
static struct ns_entry *g_namespaces = NULL;//管理所有ssd的namespace
```
在所有的ssd以及对应的namespace都注册管理完成后，进行数据读写操作。

```

static void
hello_world(void)
{
	struct ns_entry			*ns_entry;
	struct hello_world_sequence	sequence;
	int				rc;

	ns_entry = g_namespaces;
	while (ns_entry != NULL) {
	
		 //每个SSD控制器下支持多namespace，同时支持多个qpair，但应用需要确保多线程进行qpair操作时保证同时只能一个线程对同一个qpair操作
		ns_entry->qpair = spdk_nvme_ctrlr_alloc_io_qpair(ns_entry->ctrlr, NULL, 0);
		if (ns_entry->qpair == NULL) {
			printf("ERROR: spdk_nvme_ctrlr_alloc_io_qpair() failed\n");
			return;
		}

		sequence.buf = spdk_dma_zmalloc(0x1000, 0x1000, NULL);//调用spdk库进行内存分配，底层依赖于dpdk实现
		sequence.is_completed = 0;
		sequence.ns_entry = ns_entry;

		snprintf(sequence.buf, 0x1000, "%s", "Hello world!\n");

		 //将sequence.buf内容写入lba启示地址为0的位置，写入完成后回调write_complete
		rc = spdk_nvme_ns_cmd_write(ns_entry->ns, ns_entry->qpair, sequence.buf,
					    0, /* LBA start */
					    1, /* number of LBAs */
					    write_complete, &sequence, 0);
		if (rc != 0) {
			fprintf(stderr, "starting write I/O failed\n");
			exit(1);
		}
		//write_complete的回调需要通过轮询的方式判断io是否完成进行
		while (!sequence.is_completed) {
			spdk_nvme_qpair_process_completions(ns_entry->qpair, 0);
		}
		
		spdk_nvme_ctrlr_free_io_qpair(ns_entry->qpair);//释放namespace，切换到下一个ns
		ns_entry = ns_entry->next;
	}
}

```

```

static void
write_complete(void *arg, const struct spdk_nvme_cpl *completion)
{
	struct hello_world_sequence	*sequence = arg;
	struct ns_entry			*ns_entry = sequence->ns_entry;
	int				rc;

	
	spdk_dma_free(sequence->buf);//写IO完成后，释放响应buf
	sequence->buf = spdk_dma_zmalloc(0x1000, 0x1000, NULL);
	//读取刚写入的内容
	rc = spdk_nvme_ns_cmd_read(ns_entry->ns, ns_entry->qpair, sequence->buf,
				   0, /* LBA start */
				   1, /* number of LBAs */
				   read_complete, (void *)sequence, 0);
	if (rc != 0) {
		fprintf(stderr, "starting read I/O failed\n");
		exit(1);
	}
}


static void
read_complete(void *arg, const struct spdk_nvme_cpl *completion)
{
	struct hello_world_sequence *sequence = arg;

	printf("%s", sequence->buf);
	spdk_dma_free(sequence->buf);
	sequence->is_completed = 1;
}

```
本示例代码主要偏向于如何应用spdk库，至于底层的spdk实现需要深入spdk代码研究。

[参考](http://www.cnblogs.com/vlhn/p/7727016.html)

 