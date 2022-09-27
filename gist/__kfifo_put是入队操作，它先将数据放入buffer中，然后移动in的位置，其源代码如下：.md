__kfifo_put是入队操作，它先将数据放入buffer中，然后移动in的位置，其源代码如下：
```C
unsigned int __kfifo_put(struct kfifo *fifo, const unsigned char *buffer, unsigned int len)
{
      unsigned int l;

     len = min(len, fifo->size - fifo->in + fifo->out);

     /*
      * Ensure that we sample the fifo->out index -before- we
      * start putting bytes into the kfifo.
      */

     smp_mb();

     /* first put the data starting from fifo->in to buffer end */
     l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));
     memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), buffer, l);

     /* then put the rest (if any) at the beginning of the buffer */
     memcpy(fifo->buffer, buffer + l, len - l);

     /*
      * Ensure that we add the bytes to the kfifo -before-
      * we update the fifo->in index.
      */

     smp_wmb();

     fifo->in += len;

     return len;
 }
```