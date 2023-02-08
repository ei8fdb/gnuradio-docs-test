# Type of Blocks

### Introduction

To take advantage of the gnuradio framework, users will create various blocks to implement the desired data processing. There are several types of blocks to choose from:

* Synchronous Blocks (1:1)
* Decimation Blocks (N:1)
* Interpolation Blocks (1:M)
* Basic (a.k.a. General) Blocks (N:M)

### Synchronous Block

The sync block allows users to write blocks that consume and produce an equal number of items per port. A sync block may have any number of inputs or outputs. When a sync block has zero inputs, its called a source. When a sync block has zero outputs, its called a sink.

#### &#x20;C++ example

An example sync block in C++:

```cpp
// 
#include <gr_sync_block.h> 

class my_sync_block : public gr_sync_block
{
public:
  my_sync_block(...):
    gr_sync_block("my block", 
                  gr_make_io_signature(1, 1, sizeof(int32_t)),
                  gr_make_io_signature(1, 1, sizeof(int32_t)))
  {
    //constructor stuff
  }

  int work(int noutput_items,
           gr_vector_const_void_star &input_items,
           gr_vector_void_star &output_items)
  {
    //work stuff...
    return noutput_items;
  }
};
```

{% hint style="info" %}
**Some observations**\
****

* noutput\_items is the length in items of all input and output buffers
* an input signature of gr\_make\_io\_signature(0, 0, 0) makes this a source block
* an output signature of gr\_make\_io\_signature(0, 0, 0) makes this a sink block
{% endhint %}

#### Python example

An example sync block in Python:

```python
// 
class my_sync_block(gr.sync_block):
    def __init__(self):
        gr.sync_block.__init__(self,
            name = "my sync block",
            in_sig = [numpy.float32, numpy.float32],
            out_sig = [numpy.float32],
        )
    def work(self, input_items, output_items):
        output_items[0][:] = input_items[0] + input_items[1]
        return len(output_items[0])
```

The input\_items and output\_items are lists of lists. The input\_items contains a vector of input samples for every input stream, and the output\_items is a vector for each output stream where we can place items. Then length of output\_items\[0] is equivalent to the noutput\_items concept we are so familiar with from the C++ blocks.

{% hint style="info" %}
**Some observations**\
****

* The length of all input vector and all output vectors is identical
* in\_sig=None would turn this into a source block
* out\_sig=None would turn this into a sink block. In this case, use len(input\_items \[0]) since output\_items is empty!
* Unlike in C++ where we use the gr::io\_signature class, here we can just create a Python list of the I/O data sizes using numpy data types, e.g.: numpy.int8, numpy.int16, numpy.float32
{% endhint %}

### Decimation Block

The decimation block is another type of fixed rate block where the number of input items is a fixed multiple of the number of output items.

#### C++ example

```cpp
// 
#include <gr_sync_decimator.h>

class my_decim_block : public gr_sync_decimator
{
public:
  my_decim_block(...):
    gr_sync_decimator("my decim block", 
                      in_sig,
                      out_sig,
                      decimation)
  {
    //constructor stuff
  }

  //work function here...
};
```

{% hint style="info" %}
**Some observations**

****

* The gr\_sync\_decimator constructor takes a 4th parameter, the decimation factor
* The user should assume that the number of input items = noutput\_items\*decimation
{% endhint %}

#### Example in Python

```python
// 
class my_decim_block(gr.decim_block):
    def __init__(self, decim_rate):
        gr.decim_block.__init__(self,
            name="my block",
            in_sig=[numpy.float32],
            out_sig=[numpy.float32],
            decim = decim_rate)
        self.set_relative_rate(1.0/decim_rate)
        self.decimation = decim_rate

    def work(self, input_items, output_items):
        output_items[0][:] = input_items[0][0::self.decimation]
        return len(output_items[0])
     
```

{% hint style="info" %}
**Some observations**\
****

* The set\_relative\_rate call configures the input/output relationship
* To set an interpolation, use self.set\_relative\_rate(interpolation)
* The following will be true len(input\_items\[i]) = len(output\_items\[j])\*decimation
{% endhint %}

### Interpolation Block

The interpolation block is another type of fixed rate block where the number of output items is a fixed multiple of the number of input items.

#### Example in C++

```cpp
// 
#include <gr_sync_interpolator.h>

class my_interp_block : public gr_sync_interpolator
{
public:
  my_interp_block(...):
    gr_sync_interpolator("my interp block", 
                         in_sig,
                         out_sig,
                         interpolation)
  {
    //constructor stuff
  }

  //work function here...
};+
```

{% hint style="info" %}
**Some observations**\
****

* The gr\_sync\_interpolator constructor takes a 4th parameter, the interpolation factor
* The user should assume that the number of input items = noutput\_items/interpolation
{% endhint %}

#### Example in Python

```python
// 
class my_interp_block(gr.interp_block):
    def __init__(self, args):
        gr.interp_block.__init__(self,
            name="my block",
            in_sig=[numpy.float32],
            out_sig=[numpy.float32])
        self.set_relative_rate(interpolation)

    #work function here...
```

### Basic Block

The basic block provides no relation between the number of input items and the number of output items. All other blocks are just simplifications of the basic block. Users should choose to inherit from basic block when the other blocks are not suitable.

#### The Adder Block revisited as a Basic Block  in C++

```cpp
// 
#include <gr_block.h>

class my_basic_block : public gr_block
{
public:
  my_basic_adder_block(...):
    gr_block("another adder block",
             in_sig,
             out_sig)
  {
    //constructor stuff
  }

  int general_work(int noutput_items,
                   gr_vector_int &ninput_items,
                   gr_vector_const_void_star &input_items,
                   gr_vector_void_star &output_items)
  {
    //cast buffers
    const float* in0 = reinterpret_cast(input_items[0]);
    const float* in1 = reinterpret_cast(input_items[1]);
    float* out = reinterpret_cast(output_items[0]);

    //process data
    for(size_t i = 0; i < noutput_items; i++) {
      out[i] = in0[i] + in1[i];
    }

    //consume the inputs
    this->consume(0, noutput_items); //consume port 0 input
    this->consume(1, noutput_items); //consume port 1 input
    //this->consume_each(noutput_items); //or shortcut to consume on all inputs

    //return produced
    return noutput_items;
  }
};

```

{% hint style="info" %}
**Some observations**\
****

* This class overloads the general\_work() method, not work()
* The general work has a parameter: ninput\_items
  * ninput\_items is a vector describing the length of each input buffer
* Before return, general\_work must manually consume the used inputs
* The number of items in the input buffers is assumed to be noutput\_items
  * This behaviour can be altered by overloading the forecast() method but is not mandatory
{% endhint %}

#### The Adder revisted as a Basic Block in Python

```python
// 
import numpy as np
from gnuradio import gr

class my_basic_adder_block(gr.basic_block):
    def __init__(self):
        gr.basic_block.__init__(self,
            name="another_adder_block",
            in_sig=[np.float32, np.float32],
            out_sig=[np.float32])

    def general_work(self, input_items, output_items):
        #buffer references
        in0 = input_items[0][:len(output_items[0])]
        in1 = input_items[1][:len(output_items[0])]
        out = output_items[0]

        #process data
        out[:] = in0 + in1

        #consume the inputs
        self.consume(0, len(in0)) #consume port 0 input
        self.consume(1, len(in1)) #consume port 1 input


        #return produced
        return len(out)
```
