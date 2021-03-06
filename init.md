# pblk-init.c文件主要流程

## 块组织
* lun和line的关系
         
          lun 0   lun 1   lun 2   lun 3
         [blk 0] [blk 0] [blk 0] [blk 0]   line 0
         [blk 1] [blk 1] [blk 1] [blk 1]   line 1
         [ ... ] [ ... ] [ ... ] [ ... ]    ...
         [blk n] [blk n] [blk n] [blk n]   line n
         
         
## 主要在pblk_init()函数模块里面
1.     pblk = kzalloc(sizeof(struct pblk), GFP_KERNEL);  
       申请内存，初始化pblk结构体，参数GFP_KERNEL内核内存的正常分配. 可能睡眠.
       pblk->dev = dev;
	     pblk->disk = tdisk;
       pblk->state = PBLK_STATE_RUNNING;.........
       .......................................
       之后初始化pblk相关数据
  
2.     ret = pblk_luns_init(pblk, dev->luns);
       /*初始化pblk->luns的物理地址bppa,lun中坏块检查*/
       pblk->luns = kcalloc(geo->nr_luns, sizeof(struct pblk_lun), GFP_KERNEL);
       /*初始化lun相关参数*/
       ret = pblk_bb_discovery(dev, rlun);/*调用检查bad block*/

3.     ret = pblk_lines_init(pblk);
       /*初始化line管理者pblk->l_mg(主要是初始化坏块表和各个链表,链表包括free_list(读写逻辑的line管理)和gc_list
       (gc逻辑的line管理 ))*/
       l_mg->nr_lines = geo->blks_per_lun;
       l_mg->log_line = l_mg->data_line = NULL;
       l_mg->l_seq_nr = l_mg->d_seq_nr = 0;
       l_mg->nr_free_lines = 0;
       ......................初始化line管理者pblk->l_mg,line元数据管理pblk_line_meta *lm初始化
       ret = pblk_lines_alloc_metadata(pblk);//为data line分配元数据管理
       ret = pblk_alloc_line_bitmaps(pblk, line);   /* Bitmap for valid/invalid blocks */
       pblk_set_provision（） ；//设置预留百分比

4.     ret = pblk_core_init(pblk);
       /*创建几个经常使用的struct的slab内核缓存区,并创建相应的mempool,管理内核缓存,
       并创建close工作队列和bb工作队列*/
       pblk->page_pool = mempool_create_page_pool(PAGE_POOL_SIZE, 0);
       pblk->line_ws_pool = mempool_create_slab_pool()
       pblk->rec_pool = mempool_create_slab_pool()
        ........................................
 
5.     ret = pblk_l2p_init(pblk);
       /*初始化l2p map(逻辑地址到物理地址映射表)，使用的是一个trans_map维护两者的关系*/
       pblk->trans_map = vmalloc(entry_size * pblk->rl.nr_secs);  //逻辑物理转换表
       pblk_ppa_set_empty(&ppa);          //ppa地址为 ~0ull
	     pblk_trans_map_set(pblk, i, ppa);   //根据ppa设置trans——map
          
6.     ret = pblk_lines_configure(pblk, flags); //conf of lines
       line = pblk_recov_l2p(pblk)；
		   line = pblk_line_get_first_data(pblk);   /* Configure next line for user data */
              
7.     ret = pblk_writer_init(pblk);//设置写操作内核定时器，创建写操作内核线程
       setup_timer(），mod_timer(），设置定时器
       pblk->writer_ts = kthread_create(pblk_write_ts, pblk, "pblk-writer-t");//创建写操作内核线程

8.     ret = pblk_gc_init(pblk); 
       /* 创建GC内核线程，GC写操作内核线程，GC读操作内核线程，设置GC操作
       内核定时器，创建两个GC操作的工作队列,初始化gc链表*/
       .................待补充，在pblk-GC.c_md
   
9.     blk_queue_write_cache(tqueue, true, false);
10.     wake_up_process(pblk->writer_ts);   //唤醒写线程
	   
11.     return pblk;
