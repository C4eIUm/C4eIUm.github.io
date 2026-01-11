---
title: glibc2.39 free总结
description: glibc2.39 free总结
pubDate: 11 01 2025
image: /image/image1.jpg
categories:
  - Pwn
tags:
  - heap
---

1. 执行free进入_int_free函数
2. 检查,如果free的地址p大于(uintptr_t) -size或者p不是0x10对齐的，报错"free(): invalid pointer"
3. 如果根据p获取到的size，小于0x20或者size不是8字节对齐，则报错"free(): invalid size"
4. 检查下一个size的prev_inuse位
5. 进入free 进入tcache的流程

   1. 根据size获取tc_idx
   2. 如果tcache已经初始化，并且tc_idx< mp_.tcache_bins
      1. 获取p指向堆块的头,p原来指的是0x10偏移的位置,这个地址为e
      2. 如果e的key等于tcache_key（一个随机数），就进入检查
         1. 循环遍历该tcache，如果循环计数器cnt大于mp_.tcache_count每个tcache存放堆块的最大个数，报错"free(): too many chunks detected in tcache"
         2. 如果tcache里有堆块没对齐，报错"free(): unaligned chunk detected in tcache 2"
         3. 如果e等于tmp，就是存在两个相同堆块，报错"free(): double free detected in tcache 2"
      3. 如果对应tc_idx的tcache没满，放入tcache，返回
6. 进入free进入fastbin的流程

   1. 如果size，小于等于MAX_FAST，并且p的下一个堆块不是top
      1. 检查，如果下一个堆块的size小于0x10或者大于av->system_mem，则报错"free(): invalid next size (fast)"
      2. 让av->have_fastchunks变为true
      3. 根据size获取对应的fastbin的idx
      4. 获取对应fastbin的地址，fb
      5. 获取fastbin最后放入的堆块，为old
      6. 如果释放的和old是同一个堆块，报错"double free or corruption (fasttop)"
      7. 加密释放堆块的fd
      8. 放入fastbin
7. 否则如果不是mmap的堆块，进入_int_free_merge_chunk函数

   1. 根据size，获取p的nextchunk
   2. 如果p是topchunk，报错"double free or corruption (top)"
   3. 如果arena是sbrk分配的，并且nextchunk的地址>av->top加上top的size，报错"double free or corruption (out)"
   4. 如果下一个堆块的pre_inuse为0，报错"double free or corruption (!prev)"
   5. 如果下一个堆块的size<0x10或者下一个堆块的size>=av->system_mem,报错"free(): invalid next size (normal)"
   6. 进行向后合并
      1. 如果p的prev_inuse为0
      2. 获取p的prev_size,为prevsize
      3. 根据prevsize更新p为上一个堆块
      4. 如果现在的p的size，与刚刚保存的prev_size不同，报错"corrupted size vs. prev_size while consolidating"
      5. unlink现在的p
         1. 如果堆块的size位不等于（根据size找到的）下一个堆块的pre_size 报错 "corrupted size vs. prev_size"
         2. 更新fd为p的fd，bk为p的bk
         3. 如果fd->bk!=p或者bk->fd!=p报错"corrupted double-linked list"
         4. 执行fd和bk的unlink过程
            1. fd->bk = bk;
            2. bk->fd = fd;
         5. 如果这个堆块不属于small bin的大小范围,并且这个堆块的fd_nextsize不等于NULL，则会进入unlink largebin的过程
            1. 如果p->fd_nextsize->bk_nextsize != p或者p->bk_nextsize->fd_nextsize != p报错"corrupted double-linked list (not small)"
            2. 如果fd->fd_nextsize == NULL
               1. 如果p->fd_nextsize == p
                  1. fd->fd_nextsize = fd->bk_nextsize = fd;
               2. 否则
                  1. fd->fd_nextsize = p->fd_nextsize;
                  2. fd->bk_nextsize = p->bk_nextsize;
                  3. p->fd_nextsize->bk_nextsize = fd;
                  4. p->bk_nextsize->fd_nextsize = fd;
            3. 否则
               1. p->fd_nextsize->bk_nextsize = p->bk_nextsize;
               2. p->bk_nextsize->fd_nextsize = p->fd_nextsize;
8. 如果是mmap的堆块，munmap它
