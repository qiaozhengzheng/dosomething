1、查看jvm参数：

```java
java -XX:+PrintCommandLineFlags -version

-XX:InitialHeapSize=1052139072 -XX:MaxHeapSize=16834225152 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 

java version "1.8.0_60"
```

2、查看当前jvm中对象数量及内存占用情况

```java
jmap -histo[:live]
num     #instances         #bytes  class name
----------------------------------------------
   1:      28317073     1316133288  [C
   2:      26481472      635555328  java.lang.String
   3:       2004558      192437568  com.xstore.auth.domain.Res
   4:       2710823      166420440  [B
   5:        282801      114872920  [I
   6:       1190490       55624912  [Ljava.lang.Object;
   7:       1334120       42691840  java.util.HashMap$Node
   8:       1732702       41584848  java.lang.Long
   9:        724132       28965280  com.xstore.auth.domain.Category
  10:       1029876       24717024  java.util.ArrayList
  11:        229133       18595600  [Lio.netty.util.Recycler$DefaultHandle;
  12:        127506       16873408  [Ljava.util.HashMap$Node;
  13:        694306       16663344  java.lang.StringBuilder
  14:        341308       16382784  com.xstore.auth.domain.RoleHasRes
  15:        621581       14848064  [Ljava.lang.Class;
  16:        166495       14651560  java.lang.reflect.Method
  17:        772898       12366368  java.lang.Integer
  18:        234887       11274576  java.util.HashMap
  19:        140649       10126728  com.xstore.org.domain.soa.Dept
  20:        389224        9341376  java.util.Date
```

3、查看jvm信息概况

```java
jmap -heap  打印Java堆概要信息，包括使用的GC算法、堆配置参数和各代中堆内存使用情况
using thread-local object allocation.
Parallel GC with 18 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 6442450944 (6144.0MB)
   NewSize                  = 2147483648 (2048.0MB)
   MaxNewSize               = 2147483648 (2048.0MB)
   OldSize                  = 4294967296 (4096.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 1723858944 (1644.0MB)
   used     = 246733544 (235.3034439086914MB)
   free     = 1477125400 (1408.6965560913086MB)
   14.312861551623564% used
From Space:
   capacity = 211288064 (201.5MB)
   used     = 2668440 (2.5448226928710938MB)
   free     = 208619624 (198.9551773071289MB)
   1.2629393016729993% used
To Space:
   capacity = 209190912 (199.5MB)
   used     = 0 (0.0MB)
   free     = 209190912 (199.5MB)
   0.0% used
PS Old Generation
   capacity = 4294967296 (4096.0MB)
   used     = 2185108496 (2083.881851196289MB)
   free     = 2109858800 (2012.118148803711MB)
   50.8760217577219% used

49941 interned Strings occupying 5248472 bytes.
```

4、分析，老年代占比很大且一直没被回收，观察发现一次fullGc后，jvm资源会释放，并且dump出来文件分析，syncCacheLevel2Data这个方法中占用比较大，就直接看这个方法的实现

```java
		    //角色资源 MAP
			ConcurrentHashMap<String, RoleResDetailDto> roleResMap = new ConcurrentHashMap<String, RoleResDetailDto>();
            //读取缓存的所有角色 100
			Map<String, String> roleMap = this.redisService.getRoles();
			Role role = null;
			for (Map.Entry<String, String> m : roleMap.entrySet()) {
				role = JSONUtils.jsonToObject(m.getValue(), Role.class);
				//RoleResDetailDto detail = getRoleResByCode(role.getCode());
                //读取缓存中的角色资源数据 100
				RoleResDetailDto detail = this.redisService.getRoleRes(role.getCode());
				if (detail == null) {
					roleResMap.remove(role.getCode());
				} else {
					roleResMap.put(role.getCode(), detail);
				}
			}
			//写入本地缓存
			LocalCacheManager.setCacheLevel2(roleResMap);
			//修改二级缓存的版本号 为 redis 版本号
            this.redisService.setLevelTwoVersion(version1.toString());
			LOG.info("[本地缓存]成功,ip:{},version:{}" , WebUtil.getServerIp(),version1);
```

分析这个roleResMap每次量都会很大，会直接放入到老年代。基本定位问题。

5、修改代码验证；





如何做双十一备战：

1、薄弱点梳理；

2、系统隔离性梳理，mysql、redis；

3、技术预案梳理；