---
title: glibc2.39 malloc总结
description: glibc2.39 malloc总结
pubDate: 11 01 2025
image: /image/image3.jpg
categories:
  - Pwn
tags:
  - heap
  - tcache
---

1. malloc首先进入__libc\_malloc
2. 在__libc\_malloc中首先申请的是tcache bin
	1. 如果没初始化tcache就初始化
	2. 检查：
		1. 检查请求的大小对应的idx，是不是小于tcache_bins的个数
		2. 检查tcache是否不为空
		3. 检查tcache->counts\[tc_idx] 是否大于0
	3. 如果检查通过那么就从tcache中取堆块，然后返回给用户，不进入\_int_malloc
3. 进入\_int_malloc
	1. 获取0x10对齐后的大小，后简记为nb
	2. 如果av等于NULL，那么sysmalloc一个
	3.  如果nb小于等于宏定义中的MAX_FAST,进入fastbin的申请流程
		1. 根据nb，获取对应fastbin的idx
		2. 获取对应idx的fastbin的地址，fb
		3. victim等于\*fb,也就是fastbin的最后放入的一个堆块
		4. 如果victim不为NULL
			1. 检查是否0x10对齐，不对齐就报错"malloc(): unaligned fastbin chunk detected 2
			2. 还原已经xor的指针
			3. 如果victim不为NULL（一般都会执行）
				1. 根据victim的size大小获取idx，victim_idx
				2. 如果victim_idx !=idx，就会报错"malloc(): memory corruption (fast)"
				3. 如果tcache已经初始化，并且获取到的tc_idx小于tcache_bins的个数
					1. 只要对应tc_idx的tcache没满，fastbin中还有堆块，那么就会让tc_victim等于fastbin最后释放的堆块
						1. 如果tc_victim不对齐，报错"malloc(): unaligned fastbin chunk detected 3"
						2. 如果是单线程
							1. 还原xor的fastbin指针
						3. 放入对应tc_idx的tcache（原来的victim也被放入，根据头插法，会被放到所有fastbin堆块最后）
					2. 直到不满足条件结束循环
				4. 返回victim 
	4. 如果nb大于 MAX_FAST，并且属于smallbin的范围
		1. 获取nb对应smallbin的idx
		2. 获取对应idx的samllbin的地址，bin
		3. 如果smallbin不为空，同时
			1. 让victim等于smallbin尾部的堆块，也就是最先放入的堆块
			2. 获取victim的bk指向的堆块，bck
			3. 如果bck的fd指针不指向victim，报错"malloc(): smallbin double linked list corrupted"
			4. 为下一个堆块设置pre_inuse标志位
			5. 让bin的bk连上bck，让后面的fd连上bin，也就是victim从smallbin中拿出
			6. 根据nb获取对应的tcache的tc_idx
			7. 如果tcache已经初始化，并且tc_idx小于tcache_bins（其实就是在tcache的范围内，tcache_bins可以理解为管控tcache的大小范围的）
				1. 只要tcachebin没满并且smallbin不为空，不断取出循环放入tcache（同样根据头插法victim也会被放入tcache，并且是最后）
				2. 直到不满足条件，循环结束
			8. 返回用户堆块
	5. 如果上述条件不满足
		1. 根据nb，获取largebin对应的idx
		2. 如果fastbin不为空
			1. 触发malloc_consolidate，合并fastbin放入unsorted bin
	6. 初始化一些tcache的东西，为下面进入unsorted bin大循环做准备
		1. 根据nb获取对应tc_idx
		2. 如果tcache已经初始化，并且tc_idx小于tcache_bins
		3. 让tcache_nb=nb;
		4. 让return_cached = 0;标志位，标记是否直接从 tcache 返回内存
		5. 让tcache_unsorted_count = 0;  用于统计本次 \_int_malloc 从 unsorted bin 移入 tcache 的 chunk 数量
	7. 进入unsorted bin大循环，最多循环1000次
		1. 初始化计数器iters = 0
		2. 只要unsorted bin不为空
			1. victim等于unsorted bin最后放入的堆块
			2. bck = victim的bk指向的堆块
			3. 获取victim的size，size
			4. 根据size，获取下一个堆块的地址，next
			5. 然后是一大堆检查
				1. 如果该堆块victim的size<0x10或者size>arena的大小就会报错"malloc(): invalid size (unsorted)"
				2. 如果下一个堆块next的size<0x10或者size>arena的大小就会报错"malloc(): invalid next size (unsorted)"
				3. 如果next的prev_size不等于victim的size就报错"malloc(): mismatching next->prev_size (unsorted)"
				4. 如果bck的fd不等于victim，或者victim的fd不等于unsorted bin就报错"malloc(): unsorted double linked list corrupted"
				5. 如果next的prev_inuse为1，就报错"malloc(): invalid next->prev_inuse (unsorted)"
			6. 如果这个堆块属于unsorted bin中smallbin的范围，并且unsorted bin只有这一个堆块，并且这个堆块还是上一次切割剩下的堆块，也就是last remainder，并且size>nb+0x20
				1. 再次切割这个堆块
					1. remainder_size = size - nb;然后remainder = chunk_at_offset (victim, nb);此时victim等于切割下来的堆块
					2. 保存remainder到last_remainder中，last_remainder=remainder
					3. 如果切割后还属于smallbin的范围，把fd_nextsize和bk_nextsize置为NULL
					4. 给victim设置头部信息，给remainder设置头部信息，给remainder的下一个堆块写入prev_size
					5. 进行检查
						1. 检查victim是否是mmap的
						2. 检查arenna的地址
						3. 检查victim是否对齐
						4. 检查victim是否size>=0x20
						5.  检查victim的大小是否≥nb
						6. 检查victim的大小是否 \<nb+0x20
					6. 返回用户堆块
			7. 如果不切割，把victim拿出，unsorted_chunks (av)->bk = bck; bck->fd = unsorted_chunks (av);
			8. 如果victim的size等于nb
				1. 给victim的下一个堆块设置prev_inuse位
				2. 如果nb>0，并且对应idx的tcache还没满
					1. 把victim放入对应idx的tcache，然后继续循环取和放入，直到tcache满或者unsorted bin为空了
					2. 让return_cached = 1，标志一会要从tcache bin返回堆块
					3. continue直接跳出while循环，跳到 如果return_cached = 1，从tcache_get获取
				3. 否则
					1. 检查是否是mmap的如果是不进行接下来的检查，检查arenna的地址，检查是否对齐，检查是否size>=0x20,    检查victim的大小是否>=用户申请的大小，检查victim的大小是否<用户申请的大小+0x20
					2. 返回victim给用户
			9. 否则说明unsorted bin中没有合适的，把unsorted bin中的堆块放入对应大小的bin中，接下来就是这一过程，这个过程是沿着unsorted bin中的fd链的，如果第一个是0x420，第二个是0x430，第三个是0x410，那么malloc 0x430以后，0x420的堆块会进入large bin，0x430的堆块会被取出返回，0x410的堆块会呆在unsorted bin，参考上面victim的size等于nb的情况
			10. 如果victim的size属于smallbin的范围
				1. 根据size，获取对应smallbin的下标，victim_index
				2. bck等于对应victim的idx的smallbin地址
				3. fwd为从smallbin最后放入的堆块
			11.  如果victim的size属于largebin的范围(下面是更新fd_nextsize和bk_nextsize)
				1.  根据size，获取对应largebin的下标，victim_index
				2. bck等于对应victim的idx的largebin地址
				3.  fwd为从largebin最后放入的堆块（即largebin中最大 chunk ）
					1. 如果idx对应的largebin不为空
						1. 将victim的size的prev_inuse位设置为1
						2. 如果victim的size<对应largebin放入的最先一个堆块（当前large bin中最小的chunk时），那么认定victim更小，因此需要将它插入到 Large Bin 的末尾（即最小 chunk 之后），把victim找到合适位置放入largebin并更新十字链表的指针
							1. 更新fwd等于largebin的地址
							2. bck为该largebin最先的第二个放入的堆块当前 Large Bin 中第二大的 chunk
							3. 让victim的fd_nextsize指向该largebin最后放入的堆块（当前 Large Bin 中最大的 chunk），这里是循环链表的缘故，victim最小了再小就指向最大的了
							4. 让victim的bk_nextsize指向该largebin最后放入的堆块（当前 Large Bin 中最大的 chunk）的bk_nextsize
							5. 该largebin最后放入的堆块（当前 Large Bin 中最大的 chunk）的bk_nextsize指向victim
						3. 否则
							1. 只要size小于即fwd（largebin中最大 chunk ），同时大于等于该large bin中最小的chunk
								1. 一直去寻找更小的堆块，
								2. 更新fwd=fwd->fd_nextsize
							2. 直到找到刚好小于等于victim size的chunk,此时fwd为这个chunk
							3. 如果victim的size正好等于这个chunk的size
								1. 更新fwd 等于 fwd的fd，准备把victim放到fwd的fd位置
							4. 否则
								1. 则说明victim要放到fwd的bk_nextsize位置
								2. victim的fd_nextsize指针指向这个chunk
								3. victim的bk_nextsize指针指向这个chunk的bk_nextsize
								4. 检查，如果这个chunk的bk_nextsize的fd_nextsize不是这个堆块报错"malloc(): largebin double linked list corrupted (nextsize)"
								5. 这个堆块（fwd）的bk_nextsize指向victim
							5. 更新bck为fwd（此时的堆块）的bk指向的堆块
							6. 检查，如果bck的fd不等于fwd报错"malloc(): largebin double linked list corrupted (bk)"
					2. 否则
						1. 让victim的fd_nextsize和bk_nextsize都指向victim
			12.  victim->bk = bck;   victim->fd = fwd; fwd->bk = victim; bck->fd = victim，victim连接fwd和bck，它们再连接victim
		3. 让tcache_unsorted_count++  tcache_unsorted_count代表tcacge中unsorted bin数量
		4. 如果return_cached = 1并且mp_.tcache_unsorted_limit （是 tcache中unsorted bin的 限制值通常为0）> 0并且tcache_unsorted_count大于mp_.tcache_unsorted_limit
			1. 直接从tcache中拿堆块，并返回用户堆块
		5. 如果循环大于1000次，退出循环
		6. 如果return_cached = 1，从tcache_get获取，并返回
	8. 接下来准备从largebin中拿
	9. 如果nb属于large bin的范围
		1. 获取对应large bin的地址，bin
		2. 如果largebin不为空，让victim变为largebin最后放入的堆块并且如果victim的size大于等于nb（意味着肯定是可以被分配的）
			1. 让victim变为large bin中最小的
			2. 只要victim的size小于nb
				1. 就去寻找更大的
			3. 直到victim的size大于等于nb
			4. 如果victim不是largebin最小的堆块并且victim的size等于victim->fd
				1. 选择切割后放入(victim->fd)的而不是先放入的(victim)
			5. 切割，remainder_size = size - nb
			6. 让victim脱链unlink
			7. 如果切割后剩下的小于 0x20
				1. 给下一个堆块设置prev_inuse位
			8. 否则
				1. 更新remainder为切割后的堆块，victim为切割下来的堆块
				2. 获取unsorted_chunks的地址，bck
				3. 更新fwd为unsorted bin中最后放入的堆块
		3.  检查，如果fwd的bk不指向bck，报错"malloc(): corrupted unsorted chunks"
			1. 让切割剩下的remainder放入unsorted bin的头部
			2. 如果remainder属于large bin的范围
				1. 把fd_nextsize和bk_nextsize设置为NULL
			3. 为victim设置头部信息，为remainder设置头部信息，为remainder的下一个 堆块设置prev_size
			4. 常规检查
			5. 返回用户堆块
	10. 位图再次搜索，如果有可用堆块，返回用户堆块
	11. 如果上述都没能返回用户堆块，使用top chunk
		1.  victim = av->top ,获取top chunk的地址
		2. 根据victim获取size
		3. 检查，如果size>av->system_mem，报错"malloc(): corrupted top size"
		4. 如果size >=nb + 0x20
			1. 切割top chunk
			2. 设置victim的头部信息，设置remainder的头部信息，为remainder的下一个堆块设置prev_size
			3. 返回用户堆块
		5. 否则 ，查看fastbin是否存在堆块
			1. 若存在进行malloc_consolidate，让fastbin堆块合并放入unsorted bin
			2. 如果nb属于smallbin的范围，获取对应smallbin的idx
			3. 如果nb属于largebin的范围，获取对应larrgebin的idx
		 6. 否则
			 1. 就是前面都不能找到合适堆块
			 2. 直接sysmalloc
			 3. 返回用户堆块


