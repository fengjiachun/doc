<p>最近再次翻netty和disruptor的源码, 发现一些地方使用AtomicXXX.lazySet()/unsafe.putOrderedXXX系列, 以前一直没有注意lazySet这个方法, 仔细研究一下发现很有意思</p>

<p>我们拿AtomicReferenceFieldUpdater的set()和lazySet()作比较, 其他AtomicXXX类似</p>

<pre><code>public void set(T obj, V newValue) {
    // ...
    unsafe.putObjectVolatile(obj, offset, newValue);
}

public void lazySet(T obj, V newValue) {
    // ...
    unsafe.putOrderedObject(obj, offset, newValue);
}</code>

1.首先set()是对volatile变量的一个写操作, 我们知道volatile的write为了保证对其他线程的可见性会追加以下两个Fence(内存屏障)
    1)StoreStore // 在intel cpu中, 不存在[写写]重排序, 这个可以直接省略了
    2)StoreLoad  // 这个是所有内存屏障里最耗性能的
    注: 内存屏障相关参考Doug Lea大大的cookbook (http://g.oswego.edu/dl/jmm/cookbook.html)

2.Doug Lea大大又说了, lazySet()省去了StoreLoad屏障, 只留下StoreStore
  在这里 http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6275329
  把最耗性能的StoreLoad拿掉, 性能必然会提高不少(虽然不能禁止写读的重排序了保证不了可见性, 但给其他应用场景提供了更好的选择, 比如上边连接中Doug Lea举例的场景)</pre>

<p>但是但是, 在好奇心驱使下我翻了下JDK的源码(unsafe.cpp):</p>

<pre><code>// 这是unsafe.putObjectVolatile()   
UNSAFE_ENTRY(void, Unsafe_SetObjectVolatile(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jobject x_h))
    UnsafeWrapper(&quot;Unsafe_SetObjectVolatile&quot;);
    oop x = JNIHandles::resolve(x_h);
    oop p = JNIHandles::resolve(obj);
    void* addr = index_oop_from_field_offset_long(p, offset);
    OrderAccess::release();
    if (UseCompressedOops) {
        oop_store((narrowOop*)addr, x);
    } else {
        oop_store((oop*)addr, x);
    }
    OrderAccess::fence();
UNSAFE_END

// 这是unsafe.putOrderedObject()
UNSAFE_ENTRY(void, Unsafe_SetOrderedObject(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jobject x_h))
    UnsafeWrapper(&quot;Unsafe_SetOrderedObject&quot;);
    oop x = JNIHandles::resolve(x_h);
    oop p = JNIHandles::resolve(obj);
    void* addr = index_oop_from_field_offset_long(p, offset);
    OrderAccess::release();
    if (UseCompressedOops) {
        oop_store((narrowOop*)addr, x);
    } else {
        oop_store((oop*)addr, x);
    }
    OrderAccess::fence();
UNSAFE_END</code></pre>

<p>仔细看代码是不是有种被骗的感觉, 他喵的一毛一样啊. 难道是JIT做了手脚?生成汇编看看</p>

<p>生成assembly code需要hsdis插件 </p>

<p>mac平台从这里下载
    <a href="https://kenai.com/projects/base-hsdis/downloads/directory/gnu-versions">https://kenai.com/projects/base-hsdis/downloads/directory/gnu-versions</a></p>

<p>linux和windows可以从R大的[高级语言虚拟机圈子]下载 <a href="http://hllvm.group.iteye.com/">http://hllvm.group.iteye.com/</a></p>

<p>为了测试代码简单, 使用AtomicLong来测:</p>

<pre><code>// set()
public class LazySetTest {
    private static final AtomicLong a = new AtomicLong();

    public static void main(String[] args) {
        for (int i = 0; i &lt; 100000000; i++) {
            a.set(i);
        }
    }
}

// lazySet()
public class LazySetTest {
    private static final AtomicLong a = new AtomicLong();

    public static void main(String[] args) {
        for (int i = 0; i &lt; 100000000; i++) {
            a.lazySet(i);
        }
    }
}</code></pre>

<p>分别执行以下命令:</p>

<pre><code>1.export LD_LIBRARY_PATH=~/hsdis插件路径/
2.javac LazySetTest.java &amp;&amp; java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly LazySetTest</code>

<code>
// ------------------------------------------------------
// set()的assembly code片段:
0x000000010ccbfeb3: mov    %r10,0x10(%r9)
0x000000010ccbfeb7: lock addl $0x0,(%rsp)     ;*putfield value
                                            ; - java.util.concurrent.atomic.AtomicLong::set@2 (line 112)
                                            ; - LazySetTest::main@13 (line 13)
0x000000010ccbfebc: inc    %ebp               ;*iinc
                                            ; - LazySetTest::main@16 (line 12)
// ------------------------------------------------------
// lazySet()的assembly code片段:
0x0000000108766faf: mov    %r10,0x10(%rcx)    ;*invokevirtual putOrderedLong
                                            ; - java.util.concurrent.atomic.AtomicLong::lazySet@8 (line 122)
                                            ; - LazySetTest::main@13 (line 13)
0x0000000108766fb3: inc    %ebp               ;*iinc
                                            ; - LazySetTest::main@16 (line 12)</code></pre>

<p>好吧, set()生成的assembly code多了一个lock前缀的指令</p>

<p>查询IA32手册可知道, lock addl $0x0,(%rsp)其实就是StoreLoad屏障了, 而lazySet()确实没生成StoreLoad屏障</p>

<p>我想知道JIT除了将方法内联, 相同代码生成不同指令是怎么做到的</p>

-----------------------------------------------------------------------------------------------------------------
http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/6e9aa487055f/src/share/vm/classfile/vmSymbols.hpp
![putObjectVolatile](http://img1.tbcdn.cn/L1/461/1/2acc564efb86dedcc9a91efbf7bdef5240a78abf)

![putOrderedObject](http://img1.tbcdn.cn/L1/461/1/25e1f793cb7b62ea8e825ffefad18e939f802279)

putObjectVolatile与putOrderedObject都在vmSymbols.hpp的宏定义中,jvm会根据instrinsics id生成特定的指令集
putObjectVolatile与putOrderedObject生成的汇编指令不同估计是源于这里了, 继续往下看
hotspot/src/share/vm/opto/libaray_call.cpp这个类:

首先看如下两行代码:
<pre><code>case vmIntrinsics::_putObjectVolatile:        return inline_unsafe_access(!is_native_ptr,  is_store, T_OBJECT,   is_volatile);
case vmIntrinsics::_putOrderedObject:         return inline_unsafe_ordered_store(T_OBJECT);</code></pre>

再看inline_unsafe_access()和inline_unsafe_ordered_store(), 不贴出全部代码了, 只贴出重要的部分:

<pre><code>bool LibraryCallKit::inline_unsafe_ordered_store(BasicType type) {
  // This is another variant of inline_unsafe_access, differing in
  // that it always issues store-store ("release") barrier and ensures
  // store-atomicity (which only matters for "long").

  // ...
  if (type == T_OBJECT) // reference stores need a store barrier.
    store = store_oop_to_unknown(control(), base, adr, adr_type, val, type);
  else {
    store = store_to_memory(control(), adr, val, type, adr_type, require_atomic_access);
  }
  insert_mem_bar(Op_MemBarCPUOrder);
  return true;
}

---------------------------------------------------------------------------------------------------------

bool LibraryCallKit::inline_unsafe_access(bool is_native_ptr, bool is_store, BasicType type, bool is_volatile) {
  // ....

  if (is_volatile) {
    if (!is_store)
      insert_mem_bar(Op_MemBarAcquire);
    else
      insert_mem_bar(Op_MemBarVolatile);
  }

  if (need_mem_bar) insert_mem_bar(Op_MemBarCPUOrder);

  return true;
}</code></pre>

我们可以看到 inline_unsafe_access()方法中, 如果是is_volatile为true, 并且是store操作的话, 有这样的一句代码 insert_mem_bar(Op_MemBarVolatile), 而inline_unsafe_ordered_store没有插入这句代码

再继续看/hotspot/src/cpu/x86/vm/x86_64.ad的membar_volatile
<pre><code>instruct membar_volatile(rFlagsReg cr) %{
  match(MemBarVolatile);
  effect(KILL cr);
  ins_cost(400);

  format %{
    $$template
    if (os::is_MP()) {
      $$emit$$"lock addl [rsp + #0], 0\t! membar_volatile"
    } else {
      $$emit$$"MEMBAR-volatile ! (empty encoding)"
    }
  %}
  ins_encode %{
    __ membar(Assembler::StoreLoad);
  %}
  ins_pipe(pipe_slow);
%}</code></pre>


lock addl [rsp + #0], 0\t! membar_volatile指令原来来自这里

总结:
错过一些细节, 但在主流程上感觉是有一点点明白了, 有错误之处请指正


参考了以下资料:
1.http://g.oswego.edu/dl/jmm/cookbook.html
2.https://wikis.oracle.com/display/HotSpotInternals/PrintAssembly
3.http://www.quora.com/How-does-AtomicLong-lazySet-work
4.http://bad-concurrency.blogspot.ru/2012/10/talk-from-jax-london.html
