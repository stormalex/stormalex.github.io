# 中断流控层 #
内核版本3.4.2

参考资料：
<ol>
<li>深入linux内核架构
<li>http://blog.csdn.net/droidphone/article/details/7489756
</ol>

中断流控层是内核通用中断子系统中的一层，中断流控制是指合理并正确处理连续发生的中断，比如一个中断在处理中，同一个中断再次到达时该如何处理，何时屏蔽该中断，何时打开中断，何时回应中断控制器等一系列操作。该层实现了与体系和硬件无关的中断流控处理操作，它针对不同的中断电器类型（电平触发、边沿触发……），实现了对应的标准中断流控处理函数。

----------


## 1. 为什么需要中断流控层 ##
因为各种中断请求的电气特性有所不同，或者硬件的中断控制器的特性有不同，这导致了以下的处理也会有所不同：
<li> 何时对中断控制器发出ack回应
<li> mask irq和unmask irq的处理
<li> 中断控制器是否需要回应eoi(End Of Interrupt)
<li> 何时打开CPU的本地irq中断，以便允许irq的嵌套
<li> 中断数据结构的同步和保护


## 2.中断流控层的实现 ##
`irq_desc`结构中的`handle_irq`成员就是中断流控层的函数，类型是：

`typedef	void (*irq_flow_handler_t)(unsigned int irq, struct irq_desc *desc);`

设置中断流控层的接口：

`irq_set_handler(unsigned int irq, irq_flow_handler_t handle);`
`void irq_set_chip_and_handler(unsigned int irq, struct irq_chip *chip, irq_flow_handler_t handle);`


例如在2440的代码中使用以下代码设置中断流控层：
<pre>
<code>
void __init s3c24xx_init_irq(void)
{
	...
	irq_set_chip_and_handler(irqno, &s3c_irq_level_chip, handle_level_irq);
	...
}
</code>
</pre>

中断流控层在整个中断过程中的位置（红色代码，流控层设置为`handle_level_irq`的前提）：
<pre>
<code>
asm_do_IRQ()
	handle_IRQ()
		irq_enter()
		generic_handle_irq()
			generic_handle_irq_desc()
				<font color=red>desc->handle_irq(irq, desc);	//调用irq_desc结构中的handle_irq，handle_level_irq</font>
					<font color=red>raw_spin_lock(&desc->lock);</font>
					<font color=red>mask_ack_irq(desc);</font>
					<font color=red>判断IRQ_INPROGRESS</font>
					handle_irq_event(desc);	//这个函数开始时设置IRQ_INPROGRESS标志，结束时清除IRQ_INPROGRESS标志
					<font color=red>unmask_irq(desc);</font>
		irq_finish()
		irq_exit()
</code>
</pre>


## 3.通用中断子系统实现的标准流控回调函数 ##
通用中断子系统实现了以下这些标准流控回调函数(kernel/irq/chip.c)：
<li> `handle_simple_irq`  用于简易流控处理
<li> `handle_level_irq`  用于电平触发中断的流控处理
<li> `handle_edge_irq`  用于边沿触发中断的流控处理
<li> `handle_fasteoi_irq`  用于需要响应eoi的中断控制器
<li> `handle_edge_eoi_irq`  用于需要相应eoi的边沿触发的中断
<li> `handle_percpu_irq`  用于只在单一cpu响应的中断
<li> `handle_nested_irq`  用于处理使用线程的嵌套中断


## 4.`handle_level_irq` ##
该函数处理电平中断的流控操作。
电平中断的特点是，只要设备的中断请求引脚（中断线）保持在预设的触发电平，中断就会一直被请求。
所以，为了避免同一个中断被重复响应，必须在处理中断前先把中断屏蔽（mask irq）， 然后回应中断（ack irq）以便复位设备的中断请求引脚，使得打开中断屏蔽后（unmask irq）不会再次触发同一个中断，当IRQ处理完成后再打开中断屏蔽（unmask irq）。屏蔽和回应中断后，还判断了`IRQ_INPROGRESS`标志，如果`IRQ_INPROGRESS`标志被设置，说明IRQ正在被处理（可能是在别的CPU上处理），此时当前CPU立即放弃处理这个IRQ。


## 5.`handle_edge_irq` ##
该函数处理边沿触发中断的流控操作。
边沿触发的特点是，只有设备的中断请求引脚（中断线）的电平发生跳变时（由低到高或者由高到低），才会发出中断请求，因为跳变是一瞬间，不像电平触发，电平是一直保持的，所以处理不当就特别容易漏掉一次中断请求。当第一个中断请求到来时，回应中断（ack irq）使得中断能够再次到来，然后进入`handle_irq_event()`处理中断，`IRQ_INPROGRESS`标志位被设置，如果中断正在处理中第二次中断来到，那么另一个CPU就设置`IRQS_PENDING`标志位表示中断挂起，然后屏蔽中断（mask irq）、回应中断（ack irq），最后退出中断执行程序。因为是mask、ack，所以只允许挂起一次中断。
<pre>
<code>
/*
 * If we're currently running this IRQ, or its disabled,
 * we shouldn't process the IRQ. Mark it pending, handle
 * the necessary masking and go out
 */
if (unlikely(irqd_irq_disabled(&desc->irq_data) ||
		    irqd_irq_inprogress(&desc->irq_data) || !desc->action)) {	//如果中断在处理中
	if (!irq_check_poll(desc)) {
		desc->istate |= IRQS_PENDING;		//挂起中断
		mask_ack_irq(desc);					//mask、ack中断
		goto out_unlock;
	}
}
/* Start handling the irq */
desc->irq_data.chip->irq_ack(&desc->irq_data);
</code>
</pre>
在第一个CPU处理中断期间，另一个请求可能由另一个CPU响应，从上面分析可知另一个CPU响应中断只是把中断挂起并退出，当第一个CPU处理完第一次中断后会接着处理被另一个CPU挂起的中断。内核在这里设置了循环来处理这种情况，直到`IRQS_PENDING`标志无效为止。而且因为另一个CPU在响应第二次中断的时候挂起irq并且mask irq，所以在循环中要unmask irq，以便第三次中断到来时另一个CPU可以响应并挂起irq。
<pre>
<code>
do {
	if (unlikely(!desc->action)) {
		mask_irq(desc);
		goto out_unlock;
	}

	/*
	 * When another irq arrived while we were handling
	 * one, we could have masked the irq.
	 * Renable it, if it was not disabled in meantime.
	 */
	if (unlikely(desc->istate & IRQS_PENDING)) {
		if (!irqd_irq_disabled(&desc->irq_data) &&
		    irqd_irq_masked(&desc->irq_data))
			unmask_irq(desc);
	}

	handle_irq_event(desc);

} while ((desc->istate & IRQS_PENDING) &&
	 !irqd_irq_disabled(&desc->irq_data));
</code>
</pre>

`IRQS_PENDING`标志会在`handle_irq_event()`中清除。

## 6.电平中断和边沿中断的比较 ##
可以看出电平中断处理流程是为了避免对一次中断的多次响应，需要屏蔽中断；
边沿中断处理流程是为了避免漏掉中断请求，第一次处理中不需要屏蔽中断。