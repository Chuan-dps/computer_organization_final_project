# computer_organization_final_project
## 資工2B 112502035 王昱權

# 問題集

**◼(Q1) GEM5 + NVMAIN BUILD-UP (40%)**    
**◼(Q2) Enable L3 last level cache in GEM5 + NVMAIN (15%)**  
**◼(Q3) Config last level cache to  2-way and full-way associative cache and test performance (15%)**  
**◼(Q4) Modify last level cache policy based on frequency based replacement policy (15%)**  
**◼(Q5) Test the performance of write back and write through policy based on 4-way associative cache with isscc_pcm(15%)** 
# Q1
* 跟著簡報安裝   
Test command :  
./build/X86/gem5.opt configs/example/se.py -c tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --caches --l2cache --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

# Q2
* 在 gem5/config/common/Caches.py 新增L3Cache
```
class L3Cache(Cache):
    assoc = 64
    tag_latency = 32
    data_latency = 32
    response_latency =32
    mshrs = 32
    tgts_per_mshr = 24
    write_buffers = 16
```
* 在 gem5/config/common/CacheConfig.py
    * 加上L3 cache的部分
    ```
    if options.cpu_type == "O3_ARM_v7a_3":
        try:
            from cores.arm.O3_ARM_v7a import *
        except:
            print("O3_ARM_v7a_3 is unavailable. Did you compile the O3 model?")
            sys.exit(1)

        dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class = \
            O3_ARM_v7a_DCache, O3_ARM_v7a_ICache, O3_ARM_v7aL2, \
            O3_ARM_v7aWalkCache, O3_ARM_v7aL3
    else:
        dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class = \
            L1_DCache, L1_ICache, L2Cache, None, L3Cache

    ```
    * 將L2 cache連接到L3 cache(原本L2 cache連接到memory)
    ```
    if options.l2cache and options.l3cache:
        # Provide a clock for the L2 and the L1-to-L2 bus here as they
        # are not connected using addTwoLevelCacheHierarchy. Use the
        # same clock as the CPUs.
        system.l2 = l2_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l2_size,
                                   assoc=options.l2_assoc)
        system.l3 = l3_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l3_size,
                                   assoc=options.l3_assoc)

        system.tol2bus = L2XBar(clk_domain = system.cpu_clk_domain)
        system.tol3bus = L3XBar(clk_domain = system.cpu_clk_domain)

        system.l2.cpu_side = system.tol2bus.master
        system.l2.mem_side = system.tol3bus.slave

	    system.l3.cpu_side = system.tol3bus.master
        system.l3.mem_side = system.membus.slave
    ```
* 在 gem5/config/common/Options.py 中新增一條 option
```
parser.add_option("--l3cache", action="store_true")
```
* 在 gem5/src/mem/XBar.py 中新增L3XBar
```
class L3XBar(CoherentXBar):
    # 256-bit crossbar by default
    width = 32

    # Assume that most of this is covered by the cache latencies, with
    # no more than a single pipeline stage for any packet.
    frontend_latency = 1
    forward_latency = 0
    response_latency = 1
    snoop_response_latency = 1
```
* 在 gem5/src/cpu/BaseCPU.py 中引入 L3 cache，新增一個函數 : addThreeLevelCacheHierarchy
```
from XBar import L2XBar, L3XBar
...
...

def addThreeLevelCacheHierarchy(self, ic, dc, l3c, iwc = None, dwc = None):
        self.addPrivateSplitL1Caches(ic, dc, iwc, dwc)
        self.toL3Bus = L3XBar()
        self.connectCachedPorts(self.toL3Bus)
        self.l3cache = l3c
        self.toL2Bus.master = self.l3cache.cpu_side
        self._cached_ports = ['l3cache.mem_side']

```
Test command :  
./build/X86/gem5.opt configs/example/se.py -c tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

# Q3
* 在command中加入 **--l3_assoc = n** 決定L3 cache是n-way associative  (n=1代表full-way而非direct-map) 
  
Test command (2-way) :   
./build/X86/gem5.opt configs/example/se.py -c /home/brian/benchmark/quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=256kB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

Test command (full-way) :   
./build/X86/gem5.opt configs/example/se.py -c /home/brian/benchmark/quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=1 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=256kB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

* 以m5out/stats.txt中紀錄的total_miss_rate比較差異性   
system.l3.overall_misses::cpu.inst                  735                       # number of overall misses   
system.l3.overall_misses::cpu.data              1530763                       # number of overall misses   
system.l3.overall_misses::total                 1531498                       # number of overall misses   

# Q4
* 在 gem5/config/common/Caches.py的L3_cache class新增一項規則：
```
replacement_policy = Param.BaseReplacementPolicy(LFURP(),"Replacement policy")
```
Test command :   
./build/X86/gem5.opt configs/example/se.py -c /home/brian/benchmark/quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=256kB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

# Q5
* gem5預設模式就是write-back
```
do nothing
```
* write-through
* 在gem5/src/mem/cache/base.cc 的 BaseCache::access 中加入以下程式碼(1073行)
```
if (blk->isWritable()) {
    PacketPtr writeclean_pkt = writecleanBlk(blk, pkt->req->getDest(), pkt->id);
    writebacks.push_back(writeclean_pkt);
}
```
Test command :   
./build/X86/gem5.opt configs/example/se.py -c /home/brian/benchmark/multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=4 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=256kB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
