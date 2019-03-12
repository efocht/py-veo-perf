
## Performance measurement with py-veo

This notebook shows an example of performance measurement of VE kernels with py-veo and py_veosinfo.


```python
import veosinfo as vi
from veo import *
import psutil, os, signal
```

Now get Aurora node information:


```python
print vi.node_info()

```

    {'status': [0, 0], 'cores': [8, 8], 'nodeid': [1, 0], 'total_node_count': 2}


### Define build options and VEO kernel source


```python
bld = VeBuild()
bld.set_build_dir("_ve_build")
bld.set_c_src("_average", r"""
#include <stdio.h>

double average(double *a, int n)
{
    int i;
    double sum = 0;

    for (i = 0; i < n; i++)
        sum += a[i];

    return sum / (double)n;
}
""", flags="-O2 -fpic -pthread -report-all -fdiag-vector=2")
```


```python
veorun_name = bld.build_veorun(flags="-O2 -fpic -pthread", verbose=True)
#ve_so_name = bld.build_so(flags="-O2 -fpic -shared", verbose=True)
```

    /opt/nec/ve/bin/ncc -O2 -fpic -pthread -report-all -fdiag-vector=2 -c _average.c -o _average.o
    ncc: vec( 101): _average.c, line 9: Vectorized loop.
    ncc: vec( 126): _average.c, line 10: Idiom detected.: Sum
    
    ---------
    NEC C/C++ Compiler (2.1.23) for Vector Engine     Tue Mar 12 00:47:02 2019
    FILE NAME: _average.c
    
    FUNCTION NAME: average
    DIAGNOSTIC LIST
    
     LINE              DIAGNOSTIC MESSAGE
    
         9: vec( 101): Vectorized loop.
        10: vec( 126): Idiom detected.: Sum
    
    
    NEC C/C++ Compiler (2.1.23) for Vector Engine     Tue Mar 12 00:47:02 2019
    FILE NAME: _average.c
    
    FUNCTION NAME: average
    FORMAT LIST
    
     LINE   LOOP      STATEMENT
    
         4:           double average(double *a, int n)
         5:           {
         6:               int i;
         7:               double sum = 0;
         8:           
         9: V------>      for (i = 0; i < n; i++)
        10: V------           sum += a[i];
        11:           
        12:               return sum / (double)n;
        13:           }
    
    
    ---------
    compile _average -> ok
    env CFLAGS="-O2 -fpic -pthread" /opt/nec/ve/libexec/mk_veorun_static _ve_build/_average.veorun _ve_build/_average.o
    "/tmp/veorunjEKGJ6X.c", line 3: warning: typedef name has already been declared
              (with same type)
      typedef unsigned long ulong;
                            ^
    
    created specific _ve_build/_average.veorun
    



### Create VEO process, context, load VEO kernel as library


```python
otids = set([_thr.id for _thr in psutil.Process().threads()])
```


```python
nodeid = 0   # VE node ID
#print vi.cpu_info(0)
vi.metrics.ve_cpu_info_cache[nodeid] = vi.cpu_info(nodeid)

proc = VeoProc(nodeid, veorun_bin=os.getcwd() + "/" + veorun_name)
#proc = VeoProc(nodeid)
```


```python
ptids = set([_thr.id for _thr in psutil.Process().threads()])
ptids - otids
```




    {238976}




```python
ctxt = proc.open_context()
```


```python
ctids = set([_thr.id for _thr in psutil.Process().threads()])
ctxt_tid = list(ctids - ptids)[0]
print "ctxt TID:", ctxt_tid
```

    ctxt TID: 239005



```python

```


```python
#lib = proc.load_library(os.getcwd() + "/" + ve_so_name)
lib = proc.static_library()
lib.average.args_type("double *", "int")
lib.average.ret_type("double")
```

Create a numpy array filled with random numbers


```python
n = 1000000     # length of random vector: 1M elements
a = np.random.rand(n)
print("VH numpy average = %r" % np.average(a))
```

    VH numpy average = 0.500050986908445


### submit async VE function request


```python
perf_before = vi.ve_pid_perf(nodeid, ctxt_tid)
```


```python
req = lib.average(ctxt, OnStack(a), n)

avg = req.wait_result()
print("VE kernel average = %r" % avg)
```

    VE kernel average = 0.500050986908445



```python
perf_after = vi.ve_pid_perf(nodeid, ctxt_tid)
metrics = vi.calc_metrics(nodeid, perf_before, perf_after)
print metrics
```

    {'L1CACHEMISS': 0.7185894903200316, 'EFFTIME': 1.765597684867689e-05, 'USRTIME': 2.8925714285714284e-05, 'USRSEC': 0.0008074785714285714, 'VTIMERATIO': 98.38749506124061, 'MOPS': 70107.48222046622, 'AVGVL': 255.9508700102354, 'CPUPORTCONF': 0.0, 'MFLOPS': 34580.200513630974, 'ELAPSED': 1.6382958889007568, 'VLDLLCHIT': 100.0, 'VOPRAT': 98.64900249468788}



```python
del req
del lib
del proc
```


```python
etids = set([_thr.id for _thr in psutil.Process().threads()])
print "process TID:", ptids - otids
print "left over TIDs:", etids - otids
```

    process TID: set([238976])
    left over TIDs: set([238976, 239005])


**NOTE** Cleanup does not work well. No idea, yet, why. Some processes are left over. You should do:
`Kernel -> Restart & Clean up` after the next step (which cleans up the bld files).

### Cleanup VEO kernel build files


```python
bld.clear()
bld.realclean()
```


```python

```
