# Introduction

Polymorphic Types are used as the carrier of data, from one block/thread to another, for such things as stream tags and message passing interfaces. PMT data types can represent a variety of data ranging from Boolean values to dictionaries. In a sense, PMTs are a way to extend the strict typing of C++ with something more flexible. This page summarizes the most important features and points of Polymorphic Types. For an exhaustive list of PMT features check the source code, specifically the header file pmt.h.

Let's dive straight into some Python code and see how we can use PMTs: 

```
block
```

First, the pmt module is imported. We assign two values (P and P2) with PMTs using the from_long() and from_complex() calls, respectively. As we can see, they are both of the same type! This means we can pass these variables to C++ through SWIG, and C++ can handle this type accordingly.

The same code as above in C++ would look like this: 

```
#include <pmt/pmt.h>
 // [...]
 pmt::pmt_t P = pmt::from_long(23);
 std::cout << P << std::endl;
 pmt::pmt_t P2 = pmt::from_complex(gr_complex(0, 1)); 
 // Alternatively: pmt::from_complex(0, 1)
 std::cout << P2 << std::endl;
 std::cout << pmt::is_complex(P2) << std::endl;
 ```
 
 Two things stand out in both Python and C++: First, we can simply print the contents of a PMT. How is this possible? Well, the PMTs have built-in capabilities to cast their value to a string (this is not possible with all types, though). Second, PMTs know their type, so we can query that, e.g. by calling the is_complex() method.

Note: When running the above as a standalone, the compiler command will look something like g++ pmt_tutorial.cpp -I$(gnuradio-config-info --prefix)/include -lgnuradio-pmt -o pmt_tutorial 

```
 pmt::pmt_t P_int = pmt::from_long(42);
 int i = pmt::to_long(P_int);
 pmt::pmt_t P_double = pmt::from_double(0.2);
 double d = pmt::to_double(P_double);
 pmt::pmt_t P_double = pmt::mp(0.2);
```

The last row shows the pmt::mp() shorthand function. It basically saves some typing, as it infers the correct from_ function from the given type.

String types play a bit of a special role in PMTs, as we will see later, and have their own converter: 
