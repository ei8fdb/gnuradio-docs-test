# Handling Flowgraphs

### Operating a Flowgraph

The basic data structure in GNU Radio is the flowgraph, which represents the connections of the blocks through which a continuous stream of samples flows. The concept of a flowgraph is an acyclic directional graph with one or more source blocks (to insert samples into the flowgraph), one or more sink blocks (to terminate or export samples from the flowgraph), and any signal processing blocks in between.

A program must at least create a GNU Radio 'top\_block', which represents the top-most structure of the flowgraph. The top blocks provide the overall control and hold methods such as 'start,' 'stop,' and 'wait'.

The general construction of a GNU Radio application is to create a gr\_top\_block, instantiate the blocks, connect the blocks together, and then start the gr\_top\_block. The following program shows how this is done. A single source and sink are used with a FIR filter between them.

```python
//     from gnuradio import gr, blocks, filter, analog
 
    class my_topblock(gr.top_block):
        def __init__(self):
            gr.top_block.__init__(self)
 
            amp = 1
            taps = filter.firdes.low_pass(1, 1, 0.1, 0.01)
 
            self.src = analog.noise_source_c(analog.GR_GAUSSIAN, amp)
            self.flt = filter.fir_filter_ccf(1, taps)
            self.snk = blocks.null_sink(gr.sizeof_gr_complex)
 
            self.connect(self.src, self.flt, self.snk)
 
    if __name__ == "__main__":
        tb = my_topblock()
        tb.start()
        tb.wait()python
```

The 'tb.start()' starts the data flowing through the flowgraph while the 'tb.wait()' is the equivalent of a thread's 'join' operation and blocks until the gr\_top\_block is done.

An alternative to using the 'start' and 'wait' methods, a 'run' method is also provided for convenience that is a blocking start call; equivalent to the above 'start' followed by a 'wait.'

#### Latency and Throughput

By default, GNU Radio runs a scheduler that attempts to optimize throughput. Using a dynamic scheduler, blocks in a flowgraph pass chunks of items from sources to sinks. The sizes of these chunks will vary depending on the speed of processing. For each block, the number of items it can process is dependent on how much space it has in its output buffer(s) and how many items are available on the input buffer(s).

The consequence of this is that often a block may be called with a very large number of items to process (several thousand). In terms of speed, this is efficient since now the majority of the processing time is taken up with processing samples. Smaller chunks mean more calls into the scheduler to retrieve more data. The downside to this is that it can lead to large latency while a block is processing a large chunk of data.

To combat this problem, the gr\_top\_block can be passed a limit on the number of output items a block will ever receive. A block may get less than this number, but never more, and so it serves as an upper limit to the latency any block will exhibit. By limiting the number of items per call to a block, though, we increase the overhead of the scheduler, and so reduce the overall efficiency of the application.

To set the maximum number of output items, we pass a value into the 'start' or 'run' method of the gr\_top\_block:

<pre class="language-python"><code class="lang-python"><strong>//     tb.start(1000)
</strong>    tb.wait()
</code></pre>

or

```python
//     tb.run(1000)
```

Using this method, we place a global restriction on the size of items to all blocks. Each block, though, has the ability to overwrite this with its own limit. Using the 'set\_max\_noutput\_items(m)' method for an individual block will overwrite the global setting. For example, in the following code, the global setting is 1000 items max, except for the FIR filter, which can receive up to 2000 items.

```python
//     tb.flt.set_max_noutput_items(2000)
    tb.run(1000)
    
```

In some situations, you might actually want to restrict the size of the buffer itself. This can help to prevent a buffer who is blocked for data from just increasing the amount of items in its buffer, which will then cause an increased latency for new samples. You can set the size of an output buffer for each output port for every block.

{% hint style="warning" %}
This is an advanced feature in GNU Radio and should not be used without a full understanding of this concept as explained below.
{% endhint %}

To set the output buffer size of a block, you simply call:

```python
// Some code    tb.blk0.set_max_output_buffer(2000)
    tb.blk1.set_max_output_buffer(1, 2000)
    tb.start()
    print tb.blk1.max_output_buffer(0)
    print tb.blk1.max_output_buffer(1)
```

In the above example, all ports of blk0 are set to a buffer size of 2000 in _items_ (not bytes), and blk1 only sets the size for output port 1, any and all other ports use the default. The third and fourth lines just print out the buffer sizes for ports 0 and 1 of blk1. This is done after start() is called because the values are updated based on what is actually allocated to the block's buffers.

#### NOTES

1\. Buffer length assignment is done once at runtime (i.e., when run() or start() is called). So to set the max buffer lengths, the set\_max\_output\_buffer calls must be done before this.

2\. Once the flowgraph is started, the buffer lengths for a block are set and cannot be dynamically changed, even during a lock()/unlock(). If you need to change the buffer size, you will have to delete the block and rebuild it, and therefore must disconnect and reconnect the blocks.

3\. This can affect throughput. Large buffers are designed to improve the efficiency and speed of the program at the expense of latency. Limiting the size of the buffer may decrease performance.

4\. The real buffer size is actually based on a minimum granularity of the system. Typically, this is a page size, which is typically 4096 bytes. This means that any buffer size that is specified with this command will get rounded up to the nearest granularity (e.g., page size). When calling max\_output\_buffer(port) after the flowgraph is started, you will get how many items were actually allocated in the buffer, which may be different than what was initially specified.

### Reconfiguring Flowgraphs

It is possible to reconfigure the flowgraph at runtime. The reconfiguration is meant for changes in the flowgraph structure, not individual parameter settings of the blocks. For example, changing the constant in a gr::blocks::add\_const\_cc block can be done while the flowgraph is running using the 'set\_k(k)' method.

Reconfiguration is done by locking the flowgraph, which stops it from running and processing data, performing the reconfiguration, and then restarting the graph by unlocking it.

The following example code shows a graph that first adds two gr::analog::noise\_source\_c blocks and then replaces the gr::blocks::add\_cc block with a gr::blocks::sub\_cc block to then subtract the sources.

```python
// Some code 
from gnuradio import gr, analog, blocks
 import time
 
 class mytb(gr.top_block):
     def __init__(self):
         gr.top_block.__init__(self)
 
         self.src0 = analog.noise_source_c(analog.GR_GAUSSIAN, 1)
         self.src1 = analog.noise_source_c(analog.GR_GAUSSIAN, 1)
         self.add  = blocks.add_cc()
         self.sub  = blocks.sub_cc()
         self.head = blocks.head(gr.sizeof_gr_complex, 1000000)
         self.snk  = blocks.file_sink(gr.sizeof_gr_complex, "output.32fc")
 
         self.connect(self.src0, (self.add,0))
         self.connect(self.src1, (self.add,1))
         self.connect(self.add, self.head)
         self.connect(self.head, self.snk) 
 
 def main():
     tb = mytb()
     tb.start()
     time.sleep(0.01) 
 
     # Stop flowgraph and disconnect the add block
     tb.lock()
     tb.disconnect(tb.add, tb.head)
     tb.disconnect(tb.src0, (tb.add,0))
     tb.disconnect(tb.src1, (tb.add,1))
 
     # Connect the sub block and restart
     tb.connect(tb.sub, tb.head)
     tb.connect(tb.src0, (tb.sub,0))
     tb.connect(tb.src1, (tb.sub,1))
     tb.unlock()
 
     tb.wait()
 
 if __name__ == "__main__":
     main()
```

During reconfiguration, the maximum noutput\_items value can be changed either globally using the 'set\_max\_noutput\_items(m)' on the gr\_top\_block object or locally using the 'set\_max\_noutput\_items(m)' on any given block object.

A block also has a 'unset\_max\_noutput\_items()' method that unsets the local max noutput\_items value so that block reverts back to using the global value.

The following example expands the previous example but sets and resets the max noutput\_items both locally and globally.

```python
// Some code
 from gnuradio import gr, analog, blocks
 import time
 
 class mytb(gr.top_block):
     def __init__(self):
         gr.top_block.__init__(self)
 
         self.src0 = analog.noise_source_c(analog.GR_GAUSSIAN, 1)
         self.src1 = analog.noise_source_c(analog.GR_GAUSSIAN, 1)
         self.add  = blocks.add_cc()
         self.sub  = blocks.sub_cc()
         self.head = blocks.head(gr.sizeof_gr_complex, 1000000)
         self.snk  = blocks.file_sink(gr.sizeof_gr_complex, "output.32fc")
 
         self.connect(self.src0, (self.add,0))
         self.connect(self.src1, (self.add,1))
         self.connect(self.add, self.head)
         self.connect(self.head, self.snk)
 
 def main():
     # Start the gr_top_block after setting some max noutput_items.
     tb = mytb()
     tb.src1.set_max_noutput_items(2000)
     tb.start(100)
     time.sleep(0.01)
 
     # Stop flowgraph and disconnect the add block
     tb.lock()
 
     tb.disconnect(tb.add, tb.head)
     tb.disconnect(tb.src0, (tb.add,0))
     tb.disconnect(tb.src1, (tb.add,1))
 
     # Connect the sub block
     tb.connect(tb.sub, tb.head)
     tb.connect(tb.src0, (tb.sub,0))
     tb.connect(tb.src1, (tb.sub,1))
 
     # Set new max_noutput_items for the gr_top_block
     # and unset the local value for src1
     tb.set_max_noutput_items(1000)
     tb.src1.unset_max_noutput_items()
     tb.unlock()
 
     tb.wait()
 
 if __name__ == "__main__":
     main()
```
