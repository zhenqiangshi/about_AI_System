
git clone https://github.com/NVIDIA/nccl-tests.git

```
root@dkh:~/nccl-tests# ./build/all_reduce_perf -b 8 -e 128M -f 2 -g 8
# nccl-tests version 2.19.4 nccl-headers=21903 nccl-library=21903
# Collective test starting: all_reduce_perf
# nThread 1 nGpus 8 minBytes 8 maxBytes 134217728 step: 2(factor) warmup iters: 1 iters: 20 agg iters: 1 validation: 1 graph: 0 unalign: 0
#
# Using devices
#  Rank  0 Group  0 Pid 584419 on        dkh device  0 [0000:14:00] NVIDIA A100-SXM4-80GB
#  Rank  1 Group  0 Pid 584419 on        dkh device  1 [0000:19:00] NVIDIA A100-SXM4-80GB
#  Rank  2 Group  0 Pid 584419 on        dkh device  2 [0000:53:00] NVIDIA A100-SXM4-80GB
#  Rank  3 Group  0 Pid 584419 on        dkh device  3 [0000:58:00] NVIDIA A100-SXM4-80GB
#  Rank  4 Group  0 Pid 584419 on        dkh device  4 [0000:90:00] NVIDIA A100-SXM4-80GB
#  Rank  5 Group  0 Pid 584419 on        dkh device  5 [0000:96:00] NVIDIA A100-SXM4-80GB
#  Rank  6 Group  0 Pid 584419 on        dkh device  6 [0000:ce:00] NVIDIA A100-SXM4-80GB
#  Rank  7 Group  0 Pid 584419 on        dkh device  7 [0000:d4:00] NVIDIA A100-SXM4-80GB
#
#                                                              out-of-place                       in-place          
#       size         count      type   redop    root     time   algbw   busbw  #wrong     time   algbw   busbw  #wrong 
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)             (us)  (GB/s)  (GB/s)         
           8             2     float     sum      -1    34.84    0.00    0.00       0    36.18    0.00    0.00       0
          16             4     float     sum      -1    35.44    0.00    0.00       0    36.26    0.00    0.00       0
          32             8     float     sum      -1    35.58    0.00    0.00       0    35.78    0.00    0.00       0
          64            16     float     sum      -1    36.13    0.00    0.00       0    36.52    0.00    0.00       0
         128            32     float     sum      -1    36.85    0.00    0.01       0    36.81    0.00    0.01       0
         256            64     float     sum      -1    36.93    0.01    0.01       0    36.61    0.01    0.01       0
         512           128     float     sum      -1    36.53    0.01    0.02       0    36.38    0.01    0.02       0
        1024           256     float     sum      -1    36.57    0.03    0.05       0    36.51    0.03    0.05       0
        2048           512     float     sum      -1    36.58    0.06    0.10       0    36.97    0.06    0.10       0
        4096          1024     float     sum      -1    37.17    0.11    0.19       0    36.85    0.11    0.19       0
        8192          2048     float     sum      -1    36.86    0.22    0.39       0    37.03    0.22    0.39       0
       16384          4096     float     sum      -1    36.47    0.45    0.79       0    36.81    0.45    0.78       0
       32768          8192     float     sum      -1    36.74    0.89    1.56       0    36.30    0.90    1.58       0
       65536         16384     float     sum      -1    37.48    1.75    3.06       0    37.55    1.75    3.05       0
      131072         32768     float     sum      -1    40.33    3.25    5.69       0    40.84    3.21    5.62       0
      262144         65536     float     sum      -1    46.38    5.65    9.89       0    45.64    5.74   10.05       0
      524288        131072     float     sum      -1    56.27    9.32   16.31       0    56.14    9.34   16.34       0
     1048576        262144     float     sum      -1    67.31   15.58   27.26       0    65.27   16.07   28.11       0
     2097152        524288     float     sum      -1    81.12   25.85   45.24       0    80.97   25.90   45.32       0
     4194304       1048576     float     sum      -1   106.56   39.36   68.88       0   105.59   39.72   69.51       0
     8388608       2097152     float     sum      -1   161.65   51.89   90.81       0   158.47   52.93   92.63       0
    16777216       4194304     float     sum      -1   273.12   61.43  107.50       0   271.67   61.75  108.07       0
    33554432       8388608     float     sum      -1   431.40   77.78  136.11       0   432.67   77.55  135.72       0
    67108864      16777216     float     sum      -1   671.74   99.90  174.83       0   672.56   99.78  174.62       0
   134217728      33554432     float     sum      -1  1295.41  103.61  181.32       0  1284.21  104.51  182.90       0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 34.9024 
#
# Collective test concluded: all_reduce_perf
#

```