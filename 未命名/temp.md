``` Go


// mapextra 结构体，只在需要时分配

type mapextra struct {
    overflow    *[]*bmap // 存储当前 buckets 数组的所有溢出桶指针
    oldoverflow *[]*bmap // 存储 oldbuckets 数组的所有溢出桶指针
    // nextOverflow 指向一个预分配好的、可直接使用的空闲溢出桶
    nextOverflow *bmap
}






// src/runtime/map_noswiss.go
// hashGrow 负责启动一次新的 map 扩容
func hashGrow(t *maptype, h *hmap) {
	// 1. **计算新尺寸 (翻倍扩容)**
	bigger := uint8(1) // 默认将 B 增加 1，即桶数量翻倍
	if !overLoadFactor(h.count+1, h.B) {
		// 如果不是因为元素数量过多，而是因为溢出桶太多，则执行“等量扩容”
		bigger = 0 // B 不增加，桶数量不变，只是整理数据
		h.flags |= sameSizeGrow // 设置一个“等量扩容”的标志位
	}
    // 2. **挂载旧仓库**
	oldbuckets := h.buckets // 保存当前桶数组的指针，它即将成为“旧仓库”
    // 3. **分配新仓库**
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil) 
	// 分配新的桶数组
    // --- 关键的准备工作 ---
	h.B += bigger            // 更新尺寸蓝图 B
	h.flags &^= sameSizeGrow // 清除“等量扩容”标志位（如果之前设置了）
	h.oldbuckets = oldbuckets // 将旧桶数组挂载到 oldbuckets 指针上
	h.buckets = newbuckets    // 让 buckets 指针指向新分配的桶数组
	
    // 4. 重置进度条
	h.nevacuate = 0 // 初始化搬家进度为 0
	h.noverflow = 0 // 新桶数组的溢出桶数量暂时为 0
	// 如果 key 和 value 都不含指针，
	//需要特殊处理溢出桶的元数据以便GC
	if h.extra != nil && h.extra.overflow != nil {
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// 此时，map 同时拥有了 oldbuckets 和 buckets 两个数据区。数据迁移尚未开始。
}























// src/runtime/map_noswiss.go
// mapassign 是编译器为 m[key] = value 生成的函数调用
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) 
unsafe.Pointer {
    // ......
    // 1. 检查是否达到了负载因子阈值
	//- overLoadFactor: 检查元素数量是否过多(元素数/桶数 > 6.5)
	//- tooManyOverflowBuckets: 检查溢出桶是否过多
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		// 2. 触发扩容：如果任一条件满足，并且当前不在扩容状态，
		//则调用 hashGrow 开始扩容
		hashGrow(t, h)
        // ... (在扩容后，重新计算桶的位置)
	}
    // ... (后续的插入逻辑) ...
}

















    ch1 := make(chan int, 3)
    ch2 := make(chan int, 3)
    areEqual := (ch1 == ch2)
    //false








// hchan 是 Go channel 在运行时的核心数据结构
type hchan struct {
 //..//
    // --- 传送带本体 (缓冲区) ---
    buf      unsafe.Pointer // 指针，指向一个环形数组
    sendx    uint           // "下一个放入" 的位置索引
    recvx    uint           // "下一个取出" 的位置索引
    // --- 两个等候区 (等待队列) ---
    recvq    waitq  // 接收者等候区 (等待接收数据的 goroutine 队列)
    sendq    waitq  // 发送者等候区 (等待发送数据的 goroutine 队列)
    // --- 传送带控制器 (锁) ---
    lock mutex              // 互斥锁，保护 hchan 中的所有字段，
    //确保同一时间只有一个goroutine 能操作 channel
}

m[13] = LabelInfo{
  Name: "ALPHA",
  Ref:  &Meta{Source: "imported"},
}

LabelInfo{
	Name: "",
	Ref:  nil,
}


// 1️⃣ 纯值型：商品库存数量
//    key  : 商品 ID
//    value: 当前库存（整数，不含指针）
var stockLeft = map[int]int{
	101: 42,   // 商品 101 还剩 42 件
	102: 0,    // 商品 102 售罄
}

// 2️⃣ 指针作 value：在线用户会话
//    key  : 会话 token
//    value: 指向 Session 结构体的指针
type Session struct {
	UserID string
	// 还有很多运行时字段……
}
var sessions = map[string]*Session{
	"tok-abc": {UserID: "u1"},
	"tok-def": {UserID: "u2"},
}

// 3️⃣ 结构体值内含指针：订单信息
//    key  : 订单号
//    value: Order 以值存放，但内部字段含指针
type Address struct{ City string }

type Order struct {
	ID       string
	ShipTo   *Address  // 指向地址的指针 → 含指针字段
	ItemNums int
}
var orders = map[string]Order{
	"ord-001": {ID: "ord-001", ShipTo: &Address{City: "Beijing"}, ItemNums: 3},
}

// ----------------- delete 操作 -----------------

func main() {
	// ❶ 纯值型库存：runtime 仅把槽位清零，GC 不会去扫，无泄漏风险
	delete(stockLeft, 102)

	// ❷ 指针 session：runtime 把指针写成 nil，避免泄漏 *Session
	delete(sessions, "tok-def")

	// ❸ 订单：runtime 把整个 Order 区块清零，内部 ShipTo 字段也被写成 nil，
	//        防止 *Address 悬挂引用
	delete(orders, "ord-001")
}






----------------------
package main

import (
	"fmt"
	"runtime"
	"time"
)

type Profile struct {
	Bio string
}

type User struct {
	Name    string
	Profile *Profile
}

func main() {
	// map1: value 含指针
	users := make(map[string]*User, 8)

	// map2: 纯值型
	hits := make(map[int]int, 8)

	// ---- 1. 填充 -------------------------------------------------
	for i := 0; i < 4; i++ {
		p := &Profile{Bio: fmt.Sprintf("No.%d", i)}
		users[fmt.Sprintf("u%d", i)] = &User{Name: fmt.Sprintf("u%d", i), Profile: p}
		hits[i] = i * 100
	}
	fmt.Println("filled users:", len(users), "hits:", len(hits))

	// ---- 2. 删除：正确做法 --------------------------------------
	delete(users, "u1")          // 删除指针元素
	delete(hits, 2)              // 删除值元素
	// 手动 nil 清理额外指针（如果 map 外还有副本）
	// users["u0"].Profile = nil  // 示意：当你自己还握有指针副本时应置 nil

	// ---- 3. 强制 GC & 观测 --------------------------------------
	printMem("after delete, before GC")
	runtime.GC()
	printMem("after GC")

	// ---- 4. 演示“不清理指针”风险 -------------------------------
	// 再添加一个有大对象的用户，随后仅 delete 而不手动清 Profile 指针
	big := make([]byte, 8<<20) // 8 MB
	users["leak"] = &User{Name: "leak", Profile: &Profile{Bio: string(big[:8])}}

	delete(users, "leak")      // 槽位标记 emptyOne，但 Profile/Bio 里的 8 MB 仍被指针握着
	printMem("before leak GC") // 查看分配
	runtime.GC()
	printMem("after leak GC")  // 你会发现 8 MB 依旧占用，因为桶里残留指针
}

// 打印堆分配量
func printMem(tag string) {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("%-25s -> heapAlloc = %.2f MB\n", tag, float64(m.HeapAlloc)/(1<<20))
	time.Sleep(100 * time.Millisecond) // 稍等，让打印更清晰
}










type Address struct {
    City string
}

type Person struct {
    Name    string
    Address *Address
}

m := make(map[Person]string)
addr := &Address{City: "New York"}
p := Person{Name: "John", Address: addr}
m[p] = "Developer"

delete(m, p)  // 删除 m 中的键值对
















inserti -> &b.tophash[1]		
insertk -> &b.keys[1]
elem    -> &b.values[1]

inserti -> &b.tophash[0]		
insertk -> &b.keys[0]
elem    -> &b.values[0]

func mapassign(t *maptype, 
h *hmap, key unsafe.Pointer) 
unsafe.Pointer {
	// 安全检查...
	// 2.2 计算键的哈希值（含随机种子）
	hash := t.Hasher(key, uintptr(h.hash0))
	// 2.3 设置“正在写入”标志
	h.flags ^= hashWriting
	// 2.4 懒加载：若还没分配任何桶,先分配1个主桶
	if h.buckets == nil {
		h.buckets = newobject(t.Bucket) 
	}
again: //─────────────────
	// 3.1 取哈希低 B 位，得到主桶索引
	bucket := hash & bucketMask(h.B)
	// 3.2 若处于扩容期，顺手迁移此桶
	if h.growing() {
		growWork(t, h, bucket)
	}
	// 3.3 取得该桶指针
	b := (*bmap)(add(h.buckets,
	 bucket*uintptr(t.BucketSize)))
	// 3.4 提取哈希高 8 位存放到 top
	top := tophash(hash)
	// 3.5 初始化三个插入用指针
	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop: //─────────────
	//──────────────────────────────────────────────────────────
	// 4 遍历主桶 + 溢出桶，查找 key 或记首个空槽
	//──────────────────────────────────────────────────────────
	for {
		// 4.1 桶内线性扫描 8 槽
		for i := uintptr(0); i < abi.OldMapBucketCount; i++ {
			// 4.1.1 tophash 不等：不是同一哈希片段
			if b.tophash[i] != top {
				// 4.1.1.1 槽为空且尚未记录空位 → 记下
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b),
        dataOffset+i*uintptr(t.KeySize))
					elem = add(unsafe.Pointer(b),
		dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+
					           i*uintptr(t.ValueSize))
				}
		// 4.1.1.2 遇到 emptyRest：后续全空，直接退出外层循环
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue // 4.1.1.3 检查下一槽
			}
			// 4.1.2 tophash 相同 → 检查完整 key
			k := add(unsafe.Pointer(b), 
			dataOffset+i*uintptr(t.KeySize))
			// 4.1.2.1 解引用间接 key
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
			// 4.1.2.2 若 key 不等，继续
			if !t.Key.Equal(key, k) {
				continue
			}

			// 4.1.2.3 找到已有 key：必要时更新 key 内容
			if t.NeedKeyUpdate() {
				typedmemmove(t.Key, k, key)
			}
			// 4.1.2.4 锁定对应 value 槽
			elem = add(unsafe.Pointer(b),
			           dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+
			           i*uintptr(t.ValueSize))
			goto done // 4.1.2.5 结束查找进入写值阶段
		}

		// 4.2 若存在溢出桶 → 继续扫描
		ovf := b.overflow(t)
		if ovf == nil {
			break // 4.3 无溢出桶：跳出 bucketloop
		}
		b = ovf // 4.2.1 切换到下一个溢出桶
	}

	//──────────────────────────────────────────────────────────
	// 5 未找到旧键：决定是否扩容或创建溢出桶
	//──────────────────────────────────────────────────────────

	// 5.1 判断负载或溢出桶数量是否触发扩容
	if !h.growing() &&
	   (overLoadFactor(h.count+1, h.B) ||
	    tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h) // 5.1.1 开始扩容
		goto again     // 5.1.2 表地址变动，重走流程
	}

	// 5.2 若所有桶都满且没记录空槽 → 新建溢出桶
	if inserti == nil {
		// 5.2.1 分配并链入新溢出桶
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, abi.OldMapBucketCount*uintptr(t.KeySize))
	}

	//──────────────────────────────────────────────────────────
	// 6 真正写入键值对
	//──────────────────────────────────────────────────────────

	// 6.1 若 key 需间接存储，先分配堆对象再写指针
	if t.IndirectKey() {
		kmem := newobject(t.Key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	// 6.2 value 间接存储同理
	if t.IndirectElem() {
		vmem := newobject(t.Elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	// 6.3 拷贝 key 数据到槽位
	typedmemmove(t.Key, insertk, key)
	// 6.4 写入 tophash，标记此槽被占
	*inserti = top
	// 6.5 map 元素计数 +1
	h.count++

done: //───────────────────
	// 7.1 理论上仍在写：若已被清零说明并发写 → 崩溃
	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	// 7.2 清除“正在写入”标志
	h.flags &^= hashWriting

	// 7.3 若 value 为间接类型，elem 当前存指针地址，再解引用一次
	if t.IndirectElem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	// 7.4 返回 value 槽地址（编译器随后将用户值写入）
	return elem
}


func mapassign(t *maptype, 
h *hmap, 
key unsafe.Pointer) unsafe.Pointer {
    // 安全检查...
    // 1. 计算哈希值
    hash := t.Hasher(key, uintptr(h.hash0))
    // 设置写标志
    h.flags ^= hashWriting  
    // 2. 延迟初始化桶数组（懒加载）
    if h.buckets == nil {
        h.buckets = newobject(t.Bucket)
    }
again:
    // 3. 计算桶位置
    bucket := hash & bucketMask(h.B)  
    // 4. 如果正在扩容，协助扩容
    if h.growing() {
        growWork(t, h, bucket)
    } 
    // 5. 获取桶并查找位置
    b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
    top := tophash(hash) 
    var inserti *uint8     // 插入位置的tophash
    var insertk unsafe.Pointer  // key的存储位置
    var elem unsafe.Pointer     // value的存储位置 
    // 6. 查找现有key或空位置
    for {
        for i := uintptr(0); i < abi.OldMapBucketCount; i++ {
            if b.tophash[i] != top {
                // 找到空位
                if isEmpty(b.tophash[i]) && inserti == nil {
                    inserti = &b.tophash[i]
                    insertk = add(unsafe.Pointer(b),  
                    dataOffset+i*uintptr(t.KeySize))
                    elem = add(unsafe.Pointer(b), 
               dataOffset+abi.OldMapBucketCount*
             uintptr(t.KeySize)+i*uintptr(t.ValueSize))
                }
                continue
            }           
            // 7. 比较key是否存在
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
            if t.IndirectKey() {
                k = *((*unsafe.Pointer)(k))
            }
            if t.Key.Equal(key, k) {
                // 8. key已存在，更新值
                elem = add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
                goto done
            }
        }    
        // 9. 遍历溢出桶
        ovf := b.overflow(t)
        if ovf == nil {
            break
        }
        b = ovf
    }   
    // 10. 没找到现有key，插入新key  
    // 11. 检查是否需要扩容
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
        goto again
    }  
    // 12. 如果没有空位，创建新溢出桶
    if inserti == nil {
        newb := h.newoverflow(t, b)
        inserti = &newb.tophash[0]
        insertk = add(unsafe.Pointer(newb), dataOffset)
        elem = add(insertk, abi.OldMapBucketCount*uintptr(t.KeySize))
    }   
    // 13. 存储新key
    if t.IndirectKey() {
        kmem := newobject(t.Key)
        *(*unsafe.Pointer)(insertk) = kmem
        insertk = kmem
    }
    typedmemmove(t.Key, insertk, key)
    *inserti = top
    h.count++    
done:
    // 恢复标志位
    h.flags &^= hashWriting    
    // 14. 返回value指针，让调用者设置value
    return elem
}


func makemap(t *maptype, hint int, h *hmap) *hmap {
    // ... // 
    B := uint8(0)
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B
    // 分配初始桶数组
    // 当B=0时，buckets会延迟分配（懒加载）
    if h.B != 0 {
        var nextOverflow *bmap
        h.
     buckets, nextOverflow = makeBucketArray(t, h.B, nil)
        if nextOverflow != nil {
            h.extra = new(mapextra)
            h.extra.nextOverflow = nextOverflow
        }
    }
    // 返回初始化好的map
    return h
}




type bmap struct {
    tophash [8]uint8      // key的哈希值高8位
    keys [8]keytype       // 存储的key，8个连续排列
    values [8]valuetype   // 存储的value，8个连续排列
    overflow uintptr      // 指向溢出桶的指针
}



type hmap struct {
    count     int       // map中元素的个数（被内置的len()函数使用）
    flags     uint8     // 标志位，表示map的状态（如迭代器、写入状态等）
    B         uint8     // 桶数量的对数，表示有2^B个桶
    noverflow uint16    // 溢出桶的大致数量
    hash0     uint32    // 哈希种子，用于计算哈希值时添加随机性

    buckets    unsafe.Pointer // 指向2^B个桶的数组
    oldbuckets unsafe.Pointer // 扩容时指向旧的桶数组
    nevacuate  uintptr        // 扩容进度，小于此值的桶已经完成迁移
    clearSeq   uint64         // 清除序列号，用于跟踪map的清除操作
    
    extra *mapextra // 额外字段，仅在需要时使用
}


// Go map 的桶结构
type bmap struct {
    tophash [abi.OldMapBucketCount]uint8

}




// Map constants common to several packages// runtime/runtime-gdb.py:MapTypePrinter contains its own copy  
const (  
    // Maximum number of key/elem pairs a bucket can hold.  
    OldMapBucketCountBits = 3 // log2 of number of elements in a bucket.  
    OldMapBucketCount     = 1 << OldMapBucketCountBits  
    // Maximum key or elem size to keep inline (instead of mallocing per element).  
    // Must fit in a uint8.    // Note: fast map functions cannot handle big elems (bigger than MapMaxElemBytes).    OldMapMaxKeyBytes  = 128  
    OldMapMaxElemBytes = 128 // Must fit in a uint8.  
)



func runqputslow(pp *p, gp *g, h, t uint32) bool {
    //..//
    // 从本地队列取出前128个goroutine
    for i := uint32(0); i < n; i++ {
        batch[i] = pp.
        runq[(h+i)%uint32(len(pp.runq))].
        ptr()
    }
    //...//
    // 关键：把新的goroutine也加入批次
    batch[n] = gp  // 第129个就是新要放入的goroutine
    // 将129个goroutine一起放到全局队列
    q := gQueue{batch[0].guintptr(), 
    batch[n].guintptr(), int32(n + 1)}
    //..//
    return true
}
// src/runtime/proc.go ready函数
func ready(gp *g, traceskip int, next bool) {
    // ... 状态处理 ...
    
    // 步骤1：标记goroutine为runnable
    casgstatus(gp, _Gwaiting, _Grunnable)
    
    // 步骤2：放入runnext字段
    runqput(mp.p.ptr(), gp, next)
    
    // 步骤3：立即检查是否需要唤醒空闲P
    wakep()
    
    releasem(mp)
}




// src/runtime/proc.go runqget函数
func runqget(pp *p) (gp *g, inheritTime bool) {
    // 1. 首先检查runnext（最高优先级）
    next := pp.runnext
    if next != 0 && pp.runnext.cas(next, 0) {
        return next.ptr(), true  // 继承时间片
    }

    // 2. 然后从runq队列头部取
    for {
        h := atomic.LoadAcq(&pp.runqhead)
        t := pp.runqtail
        if t == h {
            return nil, false  // 队列为空
        }
        gp := pp.runq[h%uint32(len(pp.runq))].ptr()
        if atomic.CasRel(&pp.runqhead, h, h+1) {
            return gp, false  // 新的时间片
        }
    }
}










src/runtime/runtime2.go

type p struct {
	m           muintptr   // 反向链接到关联的 m (如果空闲则为 nil)
	//...//
}

type m struct {
	g0      *g // 拥有调度栈的 goroutine (g0)
	//...//
	p  puintptr  // 关联的 p，用于执行 go 代码 
	//...//
}


src/runtime/proc.go

//优先根据GOMAXPROCS的配置决定p的个数
if n, ok := strconv.Atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
    procs = n
    sched.customGOMAXPROCS = true
} else {
    //默认使用cpu核心数
    procs = defaultGOMAXPROCS(numCPUStartup)
}


src/runtime/runtime2.go

type schedt struct {
    // …… //
    pidle      puintptr       // 空闲 P 的链表头，保存可分配的工作上下文
    npidle     atomic.Int32   // 空闲 P 的数量，用于快速判断是否有可用 P
    // …… //
}



src/runtime/proc.go

// 从当前的 g 的栈，切换到当前 M 的 g0 栈，并执行 fn(g)。
func mcall(fn func(*g))



// Go 调度器本地队列（runq）与 runnext 数据结构实现及设计理念详解  
// 基于 Go 源码 src/runtime/runtime2.go 和 src/runtime/proc.go//  
// 本文件展示了 Go 语言调度器中 P（Processor）的本地运行队列设计  
// 这是 Go 高效并发调度的核心组件之一  
  
package runtime  
  
import (  
    "runtime/internal/atomic"  
    "unsafe")  
  
// ============================================================================  
// 核心数据结构定义  
// ============================================================================  
  
// guintptr 是一个原子指针类型，用于安全地存储和操作 goroutine 指针  
// 在多线程环境下保证指针操作的原子性  
type guintptr uintptr  
  
// cas 原子比较并交换操作  
// 这是实现无锁数据结构的关键原语  
func (gp *guintptr) cas(old, new guintptr) bool {  
    return atomic.Casuintptr((*uintptr)(unsafe.Pointer(gp)), uintptr(old), uintptr(new))  
}  
  
// ptr 安全地将 guintptr 转换为 *g 指针  
func (gp guintptr) ptr() *g {  
    return (*g)(unsafe.Pointer(gp))  
}  
  
// set 原子地设置指针值  
func (gp *guintptr) set(g *g) {  
    atomic.Storeuintptr((*uintptr)(unsafe.Pointer(gp)), uintptr(unsafe.Pointer(g)))  
}  
  
// g 代表一个 goroutine 的结构体（简化版本）  
type g struct {  
    // ... 其他字段  
    schedlink guintptr // 用于链接到调度队列中的下一个 goroutine    // ... 其他字段  
}  
  
// ============================================================================  
// P（Processor）结构体中的队列相关字段  
// ============================================================================  
  
type p struct {  
    id     int32  // P 的唯一标识符  
    status uint32 // P 的状态（空闲、运行中等）  
    //...//
    runqhead uint32        // 队列头部索引（消费者读取位置）  
    runqtail uint32        // 队列尾部索引（生产者写入位置）  
    runq     [256]guintptr // 环形缓冲区，存储等待运行的 goroutine 指针  
    //...//
    runnext guintptr  
    // ... 其他字段  
}  
  
// ============================================================================  
// 队列操作函数实现  
// ============================================================================  
  
// runqput 将 goroutine 添加到本地运行队列  
//  
// 参数：  
//  
//  pp: 目标 P//  gp: 要添加的 goroutine//  next: 是否放入 runnext 优先队列  
//  
// 设计理念：  
// 1. 优先使用 runnext：提供低延迟调度  
// 2. 环形缓冲区：高效的队列实现  
// 3. 自动降级：本地队列满时转移到全局队列  
func runqput(pp *p, gp *g, next bool) {  
    // 随机化调度顺序，用于竞态检测时发现调度假设错误  
    if randomizeScheduler && next && randn(2) == 0 {  
       next = false  
    }  
  
    // ========================================================================  
    // runnext 优先队列处理  
    // ========================================================================  
    if next {  
    retryNext:  
       // 原子地获取当前 runnext 的值  
       oldnext := pp.runnext  
  
       // 尝试原子地将新 goroutine 设置为 runnext       // 这里使用 CAS 操作保证线程安全  
       if !pp.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {  
          // CAS 失败，可能是其他线程修改了 runnext，重试  
          goto retryNext  
       }  
  
       if oldnext == 0 {  
          // 之前 runnext 为空，直接设置成功  
          return  
       }  
  
       // runnext 之前有值，将旧值踢到常规队列中  
       // 这确保了 runnext 只有一个槽位，新的优先级更高  
       gp = oldnext.ptr()  
    }  
  
    // ========================================================================  
    // 常规环形队列处理  
    // ========================================================================  
retry:  
    // 使用 load-acquire 语义读取队列头部，与消费者同步  
    h := atomic.LoadAcq(&pp.runqhead)  
    t := pp.runqtail  
  
    // 检查队列是否还有空间  
    if t-h < uint32(len(pp.runq)) {  
       // 队列未满，将 goroutine 添加到尾部  
       // 使用模运算实现环形缓冲区  
       pp.runq[t%uint32(len(pp.runq))].set(gp)  
  
       // 使用 store-release 语义更新尾部索引，使元素对消费者可见  
       atomic.StoreRel(&pp.runqtail, t+1)  
       return  
    }  
  
    // 队列已满，需要将一部分工作转移到全局队列  
    if runqputslow(pp, gp, h, t) {  
       return  
    }  
  
    // 慢路径处理完成，队列现在应该有空间了，重试  
    goto retry  
}  
  
// runqputslow 处理本地队列满时的情况  
// 将本地队列的一半 goroutine 转移到全局队列  
//  
// 设计理念：  
// 1. 负载均衡：防止某个 P 积累过多工作  
// 2. 工作窃取：其他空闲的 P 可以从全局队列获取工作  
// 3. 批量操作：减少全局队列的锁竞争  
func runqputslow(pp *p, gp *g, h, t uint32) bool {  
    // 创建批处理数组，大小为队列一半加上当前要添加的 goroutine    var batch [len(pp.runq)/2 + 1]*g  
  
    // 计算要转移的 goroutine 数量（队列的一半）  
    n := t - h  
    n = n / 2  
  
    if n != uint32(len(pp.runq)/2) {  
       throw("runqputslow: queue is not full")  
    }  
  
    // 从本地队列头部取出一半的 goroutine    for i := uint32(0); i < n; i++ {  
       batch[i] = pp.runq[(h+i)%uint32(len(pp.runq))].ptr()  
    }  
  
    // 原子地更新队列头部，提交消费操作  
    if !atomic.CasRel(&pp.runqhead, h, h+n) {  
       // CAS 失败，可能是有其他线程在窃取工作，返回 false 重试  
       return false  
    }  
  
    // 将当前要添加的 goroutine 也加入批次  
    batch[n] = gp  
  
    // 可选的随机化，用于测试  
    if randomizeScheduler {  
       for i := uint32(1); i <= n; i++ {  
          j := cheaprandn(i + 1)  
          batch[i], batch[j] = batch[j], batch[i]  
       }  
    }  
  
    // 将 goroutine 链接成链表，准备添加到全局队列  
    for i := uint32(0); i < n; i++ {  
       batch[i].schedlink.set(batch[i+1])  
    }  
  
    // 创建队列结构，包含头、尾和大小信息  
    q := gQueue{  
       head: batch[0].guintptr(),  
       tail: batch[n].guintptr(),  
       size: int32(n + 1),  
    }  
  
    // 将批次添加到全局队列（需要获取全局锁）  
    lock(&sched.lock)  
    globrunqputbatch(&q)  
    unlock(&sched.lock)  
  
    return true  
}  
  
// runqget 从本地运行队列获取可执行的 goroutine//  
// 返回值：  
//  
//  gp: 获取到的 goroutine//  inheritTime: 是否继承当前时间片  
//  
// 调度优先级：  
// 1. 首先检查 runnext 优先队列  
// 2. 然后从常规环形队列头部取出  
func runqget(pp *p) (gp *g, inheritTime bool) {  
    // ========================================================================  
    // 优先检查 runnext 队列  
    // ========================================================================  
  
    // 获取 runnext 中的 goroutine    next := pp.runnext  
  
    // 如果 runnext 非空，尝试原子地将其取出  
    // 注意：如果 CAS 失败，说明被其他 P 窃取了，不需要重试  
    // 因为其他 P 只能将 runnext 设为 0，只有当前 P 能设为非零值  
    if next != 0 && pp.runnext.cas(next, 0) {  
       // 成功取出 runnext，返回 true 表示继承时间片  
       // 这是 runnext 优化的关键：减少上下文切换开销  
       return next.ptr(), true  
    }  
  
    // ========================================================================  
    // 从常规环形队列获取  
    // ========================================================================  
  
    for {  
       // 使用 load-acquire 语义读取队列头部，与其他消费者同步  
       h := atomic.LoadAcq(&pp.runqhead)  
       t := pp.runqtail  
  
       // 检查队列是否为空  
       if t == h {  
          return nil, false  
       }  
  
       // 从队列头部取出 goroutine       gp := pp.runq[h%uint32(len(pp.runq))].ptr()  
  
       // 原子地更新队列头部索引，提交消费操作  
       if atomic.CasRel(&pp.runqhead, h, h+1) {  
          // 成功获取，返回 false 表示开始新的时间片  
          return gp, false  
       }  
       // CAS 失败，可能是其他线程也在消费，重试  
    }  
}  
  
// runqempty 检查本地队列是否为空  
// 需要同时检查 runnext 和环形队列，并处理可能的竞态条件  
func runqempty(pp *p) bool {  
    // 需要防止竞态条件：  
    // 1. P 在 runnext 有 G1，但 runqhead == runqtail    // 2. runqput 将 G1 踢到 runq 中  
    // 3. runqget 清空了 runnext    // 简单地观察 runqhead == runqtail 和 runnext == nil 并不能保证队列真的为空  
  
    for {  
       head := atomic.Load(&pp.runqhead)  
       tail := atomic.Load(&pp.runqtail)  
       runnext := atomic.Loaduintptr((*uintptr)(unsafe.Pointer(&pp.runnext)))  
  
       // 重新检查 runqtail，确保读取的一致性  
       if tail == atomic.Load(&pp.runqtail) {  
          return head == tail && runnext == 0  
       }  
       // 读取不一致，重试  
    }  
}  
  
// ============================================================================  
// 工作窃取相关函数  
// ============================================================================  
  
// runqsteal 从其他 P 的本地队列窃取工作  
// 这是 Go 调度器实现负载均衡的重要机制  
//  
// 参数：  
//  
//  pp: 目标 P（窃取者）  
//  p2: 源 P（被窃取者）  
//  stealRunNextG: 是否可以窃取 runnext//  
// 设计理念：  
// 1. 负载均衡：忙碌的 P 分担工作给空闲的 P// 2. 局部性保持：只窃取一半，保持原 P 的缓存局部性  
// 3. 最小化竞争：通过 CAS 操作实现无锁窃取  
func runqsteal(pp, p2 *p, stealRunNextG bool) *g {  
    t := pp.runqtail  
  
    // 尝试从 p2 窃取 goroutine 到 pp 的队列尾部  
    n := runqgrab(p2, &pp.runq, t, stealRunNextG)  
    if n == 0 {  
       return nil // 没有窃取到任何工作  
    }  
  
    // 窃取成功，返回其中一个 goroutine 立即执行  
    n--  
    gp := pp.runq[(t+n)%uint32(len(pp.runq))].ptr()  
  
    if n == 0 {  
       // 只窃取到一个，直接返回  
       return gp  
    }  
  
    // 窃取到多个，更新队列尾部索引，使其他的可用于后续调度  
    h := atomic.LoadAcq(&pp.runqhead)  
    if t-h+n >= uint32(len(pp.runq)) {  
       throw("runqsteal: runq overflow")  
    }  
    atomic.StoreRel(&pp.runqtail, t+n)  
  
    return gp  
}  
  
// runqgrab 执行实际的窃取操作  
// 从源队列抓取约一半的 goroutine 到目标批次中  
func runqgrab(pp *p, batch *[256]guintptr, batchHead uint32, stealRunNextG bool) uint32 {  
    for {  
       // 原子地读取源队列的头尾索引  
       h := atomic.LoadAcq(&pp.runqhead)  
       t := atomic.LoadAcq(&pp.runqtail)  
  
       // 计算要窃取的数量（约一半）  
       n := t - h  
       n = n - n/2  
  
       if n == 0 {  
          // 常规队列为空，尝试窃取 runnext          if stealRunNextG {  
             if next := pp.runnext; next != 0 {  
                // 如果源 P 正在运行，稍等一下避免干扰其调度 runnext                if pp.status == _Prunning {  
                   if !osHasLowResTimer {  
                      usleep(3) // 等待约 3 微秒  
                   } else {  
                      osyield() // 让出 CPU                   }  
                }  
  
                // 尝试原子地窃取 runnext                if !pp.runnext.cas(next, 0) {  
                   continue // CAS 失败，重试  
                }  
  
                batch[batchHead%uint32(len(batch))] = next  
                return 1  
             }  
          }  
          return 0 // 没有可窃取的工作  
       }  
  
       // 检查读取的一致性  
       if n > uint32(len(pp.runq)/2) {  
          continue // 读取不一致，重试  
       }  
  
       // 复制 goroutine 到批次数组  
       for i := uint32(0); i < n; i++ {  
          g := pp.runq[(h+i)%uint32(len(pp.runq))]  
          batch[(batchHead+i)%uint32(len(batch))] = g  
       }  
  
       // 原子地更新源队列头部，提交窃取操作  
       if atomic.CasRel(&pp.runqhead, h, h+n) {  
          return n // 窃取成功  
       }  
       // CAS 失败，重试  
    }  
}  
  
// ============================================================================  
// 设计理念总结  
// ============================================================================  
  
/*  
Go 调度器本地队列设计的核心理念：  
  
1. 【性能优化】  
   - 无锁设计：使用原子操作和 CAS 避免锁竞争  
   - 缓存友好：连续内存布局和局部性优化  
   - 批量操作：减少全局队列访问频率  
  
2. 【并发安全】  
   - 原子操作：保证多线程环境下的数据一致性  
   - 内存屏障：使用 acquire/release 语义确保可见性  
   - ABA 问题处理：通过精心设计的 CAS 操作避免  
  
3. 【负载均衡】  
   - 工作窃取：空闲 P 可以从忙碌 P 窃取工作  
   - 全局降级：本地队列满时转移到全局队列  
   - 公平调度：定期检查全局队列确保公平性  
  
4. 【延迟优化】  
   - runnext 优先队列：为相关 goroutine 提供低延迟调度  
   - 时间片继承：减少上下文切换开销  
   - 局部性利用：提高缓存命中率  
  
5. 【可扩展性】  
   - 分布式设计：每个 P 独立的本地队列  
   - 最小化竞争：大部分操作在本地完成  
   - 动态适应：根据负载情况自动调整  
  
这种设计使得 Go 能够高效地调度大量的 goroutine，  
实现了高并发、低延迟和良好可扩展性的完美平衡。  
*/


type p struct {
    id     int32  // P 的唯一标识符
    status uint32 // P 的状态（空闲、运行中等）
    runqhead uint32        // 队列头部索引（消费者读取位置）
    runqtail uint32        // 队列尾部索引（生产者写入位置）
    runq     [256]guintptr // 环形缓冲区，存储等待运行的 goroutine 指针
    runnext guintptr
    // ... 其他字段
}


func currGoroutineDo (){
    go func() { // <-- 在这行代码的背后，一个_Gidle的G被初始化并变为_Grunnable
    fmt.Println("Hello from a new goroutine!")
    }()
}




// src/runtime/proc.go findRunnable函数片段
func findRunnable() (gp *g, inheritTime, tryWakeP bool) {
//...//
if pp.schedtick%61 == 0 && !sched.runq.empty() {
    // %61 == 0: 每61次调度检查一次，这是一个经验值，用于平衡性能和公平性
    lock(&sched.lock)           // 获取调度器全局锁，保护全局队列操作
    gp := globrunqget()         // 从全局队列获取一个goroutine
    unlock(&sched.lock)         // 释放调度器全局锁
    if gp != nil {
        // 成功获取到goroutine，立即返回执行
        // 第二个参数false: 不继承时间片，使用完整的时间片
        // 第三个参数false: 不需要唤醒额外的P
        return gp, false, false
    }
    }
    //...//
    }
```
