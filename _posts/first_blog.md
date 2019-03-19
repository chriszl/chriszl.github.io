### һ SPDK���
SPDK��intel��˾ΪNVME ssdӲ��������һ�����ڼ���Ӳ�����ܵ�һ����׼���֧�ֲ�ͬ�����lib�⣬����nvme ssd driver��ioat��bdev�ȡ�
###�� Դ��Ŀ¼

 - app
	 - app/iscsi_tgt: iscsi target
	 - app/nvmf_tgt:  NVMe-oF target
	 - app/iscsi_top��iscsi top���� ������linux top���������iscsi
	 - app/trace��iscsi target��nvme-of target  trace����
	 - app/vhost����virtio���������ָ�����qemu�������������IO���д���
 - build
 - doc��spdk �µ�doc�ļ�
 - dpdk��spdk������dpdk�ĺܶ������
 - etc��������ʹ�÷�ʽ�Ļ�������
 - examples��ʾ������
 - lib��������
 - mk��makefile�ļ�
 - scripts���ű��������������
 - test����ģ�鹦�����ܲ���
 
 ###�� ʾ������
 ������Ҫ��hello_world���뿪ʼ������Ӧ�����ʹ��nvme ssd�豸��hello_world��������ssd�洢���ݲ�������
 

```
int main(int argc, char **argv){
	int rc;
	struct spdk_env_opts opts;

	spdk_env_opts_init(&opts);
	opts.name = "hello_world";
	opts.shm_id = 0;
	if (spdk_env_init(&opts) < 0) {//ǰ�벿�ִ�����Ҫ����������ʼ��spdk���������û�����
		fprintf(stderr, "Unable to initialize SPDK env\n");
		return 1;
	}

	printf("Initializing NVMe Controllers\n");

	rc = spdk_nvme_probe(NULL, NULL, probe_cb, attach_cb, NULL);//���ýӿڽ������ڵ�nvme ssd�����б�
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
	hello_world();//��д����
	cleanup();//ֹͣattach ��Ӧ��nvme ssd
	return 0;
}

```
��Ҫ���̣�

 1. ��ʼ��spdk����
 2. ɨ��nvme ssd������
 3. ��д����
 4. ж��nvme ssd
���У�����nvme ssd����������ڶ��ssd����ι���

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

	//namespace==scis lun����ͬ��ssd֧�ֵ�namespace������һ��
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
���ѷ��֣�probe_cb��ɨ���Ӧ��nvme ssd�豸����лص���attach_cb�ڱ����غ�ص�������nvme ssd���ڶ��namespace���൱�ڻ�еӲ�̵�LUN�������Ҫ���й��������������������ݽṹ��

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

static struct ctrlr_entry *g_controllers = NULL;//�������е�nvme ssd
static struct ns_entry *g_namespaces = NULL;//��������ssd��namespace
```
�����е�ssd�Լ���Ӧ��namespace��ע�������ɺ󣬽������ݶ�д������

```

static void
hello_world(void)
{
	struct ns_entry			*ns_entry;
	struct hello_world_sequence	sequence;
	int				rc;

	ns_entry = g_namespaces;
	while (ns_entry != NULL) {
	
		 //ÿ��SSD��������֧�ֶ�namespace��ͬʱ֧�ֶ��qpair����Ӧ����Ҫȷ�����߳̽���qpair����ʱ��֤ͬʱֻ��һ���̶߳�ͬһ��qpair����
		ns_entry->qpair = spdk_nvme_ctrlr_alloc_io_qpair(ns_entry->ctrlr, NULL, 0);
		if (ns_entry->qpair == NULL) {
			printf("ERROR: spdk_nvme_ctrlr_alloc_io_qpair() failed\n");
			return;
		}

		sequence.buf = spdk_dma_zmalloc(0x1000, 0x1000, NULL);//����spdk������ڴ���䣬�ײ�������dpdkʵ��
		sequence.is_completed = 0;
		sequence.ns_entry = ns_entry;

		snprintf(sequence.buf, 0x1000, "%s", "Hello world!\n");

		 //��sequence.buf����д��lba��ʾ��ַΪ0��λ�ã�д����ɺ�ص�write_complete
		rc = spdk_nvme_ns_cmd_write(ns_entry->ns, ns_entry->qpair, sequence.buf,
					    0, /* LBA start */
					    1, /* number of LBAs */
					    write_complete, &sequence, 0);
		if (rc != 0) {
			fprintf(stderr, "starting write I/O failed\n");
			exit(1);
		}
		//write_complete�Ļص���Ҫͨ����ѯ�ķ�ʽ�ж�io�Ƿ���ɽ���
		while (!sequence.is_completed) {
			spdk_nvme_qpair_process_completions(ns_entry->qpair, 0);
		}
		
		spdk_nvme_ctrlr_free_io_qpair(ns_entry->qpair);//�ͷ�namespace���л�����һ��ns
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

	
	spdk_dma_free(sequence->buf);//дIO��ɺ��ͷ���Ӧbuf
	sequence->buf = spdk_dma_zmalloc(0x1000, 0x1000, NULL);
	//��ȡ��д�������
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
��ʾ��������Ҫƫ�������Ӧ��spdk�⣬���ڵײ��spdkʵ����Ҫ����spdk�����о���

[�ο�](http://www.cnblogs.com/vlhn/p/7727016.html)

 