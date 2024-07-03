#### MYDB

##### 0项目结构

1. 实现功能：数据可靠性和数据恢复；MVCC；两种事务隔离级别；死锁处理；简单的表和字段管理；简单的SQL查询；基于socket的Server和Client
2. MYDB分为5个模块：
   1. transaction manager(TM)，通过维护XID文件来维护事务状态，并提供接口供其他模块查询某个事务的状态
   2. data manager(DM)，管理数据库DB文件和日志文件
   3. version manager(VM)，基于两阶段锁协议实现调度序列的可串行化，并实现mvcc以消除读写阻塞，并实现两种隔离级别
   4. index manager(IM)，基于B+树的索引
   5. Table manager(TBM)，实现对字段和表的管理
3. insert语句执行流程：
   1. TBM接受语句, 并进行解析.
   2. TBM将values的值二进制化.
   3. TBM利用VM的insert操作, 将二进制化后的数据, 插入到数据库文件.
   4. VM为该条数据建立版本控制, 并利用DM的insert操作, 将数据插入到数据库.
   5. DM将数据插入到数据库, 并返回其被存储的地址.
   6. VM将得到的地址, 作为该条记录的handler, 返回给TBM.
   7. TBM计算该条语句的key, 并将handler作为data, 并调用IM的insert, 建立索引.
   8. IM利用DM提供的read和insert等操作, 将key和data存入索引中.
   9. TBM返回客户端插入成功的信息.
4. 

#####  1TM

1. XID文件：每个事务都有一个XID(自增，从1开始)，XID头部8字节记录事务总个数，并给每个事务分配一个字节空间保存状态(active, committed, aborted)，因此事务 xid 在文件中的状态就存储在 (xid-1)+8 字节处，xid-1 是因为 xid 0（Super XID） 的状态不需要记录
2. 文件读写采用Java NIO方式的FileChannel配合ByteBuffer
3. ByteBuffer是Java NIO库中的一个类，用于处理字节数据，提供一种灵活高效的方式来操作字节缓冲区，适用于处理大量的字节数据，如文件IO，网络通信；
4. Position，limit：容量为10的ByteBuffer中写入5个字节的数据，写模式下，limit=10，position=5；读模式下，limit=5
5. allocate创建新的、独立的缓冲区，wrap适用于共享已有字节数组的缓冲区
6. FileChannel  读fc.position(offset)   fc.read(buffer)   写fc.position(offset)  fc.write(buffer)  fc.force(false)强制将文件通道中所有未写入的数据写入到磁盘(buffer->channel->磁盘)
7. 总的来说，TM的实现就是根据XID文件(头部和事务的xid)，实现checkXIDCounter方法，begin方法，updateXid方法，incrXIDCounter方法，checkXID方法；管理事务的开始，提交和回滚
8. XID自增

##### 2DM 引用计数缓存框架和共享内存数组

1. DM直接管理数据库DB文件和日志文件，是上层模块和文件系统之间的一个抽象层，向下读写文件向上提供数据的包装，且都提供了一个缓存的功能，用内存操作来保证效率
2. 引用计数缓存框架：与LRU相比，缓存的释放是由上层模块主动调用释放方法来触发的，而不是被动地由缓存管理器自动驱逐，只有当资源的引用计数归零时，缓存才会驱逐该资源，这种方式可以确保缓存中的资源只有在确实不再被使用时才会被释放，避免了不必要的资源驱逐和回源操作
3. 引用计数缓存框架AbstractCache，定义两个抽象方法getForCache(key),   releaseForCache(obj)在子类实现；  定义三个HashMap，cache实际缓存的数据，references元素的引用个数，getting(boolean)是否正在获取某资源；主要实现get, release, close方法
4. get方法，release方法，close方法(把缓存中所有资源都释放)，都是局部上锁lock.lock()，碰见不同情况会各自unlock，配合try  catch；主要就是操作这三个HashMap
5. 共享内存数组：Java是把数组作为一个对象存储而非指针，因此无法共享内存数组，只能自定义同一个数组，加上start,  end模仿共享内存

##### 3DM 数据页的缓存与管理

1. 针对DM模块向下对文件系统的抽象部分，将文件系统抽象成页面，每次对文件系统的读写都是以页面为单位进行缓存；页面是存储在内存中的数据单元
2. 页面：定义页面Page(pageNumber, data, dirty, lock,  包含PageCache，,能够快速对页面缓存进行释放操作)
3. 页面中包含页面缓存，就是继承于引用计数缓存框架，能够向上提供get, release, close接口供上层使用，page的data包含了实际的字节数据，PageCache就是提供接口操作page的数据实现缓存的功能，缓存以Page为单位进行缓存
4. 页面缓存接口PageCache(包括新建，获取，释放，关闭截断，刷新页面等方法)，页面缓存需要继承抽象缓存框架AbstractCache，需要实现具体的getForCache方法(用于从文件读取页面数据，并包裹成Page返回)，releaseForCache方法(用于驱逐页面时决定是否将脏数据协会到文件系统)
5. getForCache直接从文件中读取(channel, buffer)，并包裹成Page即可；releaseForCache判断页面是否是脏页面，来决定是否需要写回文件系统
6. 数据页管理：数据库文件第一页常用于做一些特殊用途，如存储一些元数据或启动检查；MYDB做启动检查，启动时会生成一串随机字节存储，关闭时会拷贝在另一位置，每次启动时检查两处的字节是否相同，用来判断上次是否自动关闭，是否需要数据的恢复流程
7. 对于普通页的管理，一个普通页面以一个2字节无符号数起始，表示这一页的空闲位置的偏移，剩下的部分都是实际存储的数据，所以读写利用System.arraycopy来操作
8. pageNumber从1开始计数，也是自增

##### 4DM 日志文件与恢复策略

1. MYDB提供崩溃后的数据恢复功能，DM层在每次对底层数据操作时，都会记录一条日志到磁盘上，在数据崩溃后再次启动，可以恢复数据文件，保证其一致性

2. 日志的二进制文件，按照如下的格式进行排布：

   ```
   [XChecksum][Log1][Log2][Log3]...[LogN][BadTail]
   ```

   每条日志的格式如下：(一条日志对应一条命令)

   ```
   [Size][Checksum][Data]
   [0, 0, 0, 3] [3, -112, -4, 93] [97, 97, 97]
   [0, 0, 0, 3] [14, 40, -23, -38] [98, 98, 98]
   [0, 0, 0, 3] [24, -64, -41, 87] [99, 99, 99]
   ```

   size和Checksum都是int整数转为byte的四字节数组

   ```
   xCheck = xCheck * SEED + b; //seed = 13331
   ```

3. create方法，init方法，checkAndRemoveTail方法，next方法，internNext方法，log方法；主要也是检查文件校验和和长度

4. 恢复策略：在进行I和U操作之前，必须先进行对应的日志操作，在保证日志写入磁盘后，才进行数据操作；

5. 两条规则：1正在进行的事务不会读取 / 修改其他未提交事务产生的数据，两条规则保证了在多线程下的恢复性；

6. MYDB日志的恢复也分为两种

   1. **通过**`**redo log**`**重做所有崩溃时已经完成（**`**committed 或 aborted**`**）的事务**
   2. **通过**`**undo log**`**撤销所有崩溃时未完成（**`**active**`**）的事务 **

7. 日志的格式类型分为insert和update，redo是正序执行日志操作，undo是反向撤销日志操作

8. redoTranscations重做所有已完成的事务，对已提交的事务产生的日志进行1判断类型2重做操作；undoTranscations撤销所有未完成的事务，对未提交的事务产生的日志，先将日志记录添加到对应的日志列表中(Map<XID, List<byte[日志]>，再继续撤销更新/插入操作

9. 

##### 5DM 页面索引与DM的实现

1. 页面索引，缓存了每一页的空闲空间；旨在提高数据库的插入操作效率，通过缓存页面的空闲空间信息，避免了频繁地访问磁盘或者缓存中的页面，从而加速了插入操作的执行。

2. PageIndex，List[] lists，       lists[number].add(new PageInfo(pgno, freespace))

3. PIndex.add(PageNumber, freespace)

4. DataItem提供了一种上层模块和底层数据存储之间进行交互的接口，功能：1数据存储和访问；2数据修改和事务管理；3数据共享和内存管理(数据通过SubArray返回给上层模块)；4缓存管理；

5. ```
   Page是对文件系统的抽象，包含了PageCache继承缓存框架，提供对上层的接口用来get/release
   
   DataItem等于在page的基础上再提供了一层封装？？？？？此外还包括原始数据/旧的原始数据，DataManagerImpl，uid  作用：除了原有的数据存储和访问和缓存管理，还有数据修改和事务管理，数据共享和内存管理
   
   此外DataManager还包括管理日志的功能，用来数据恢复，所以还提供了创建日志和根据undo和redo log进行数据恢复或者撤销的功能
   ```

6. 

7. DataItem中保存的数据，结构为

   ```
   [ValidFlag] [DataSize] [Data]
   ```

   ValidFlag占用1字节，标识是否有效，删除一个DataItem只需要置为0；DataSize占用2字节

   ```java
   public class DataItemImpl implements DataItem {
       private SubArray raw; //原始数据
       private byte[] oldRaw; //旧的原始数据
       private DataManagerImpl dm; //数据管理器
       private long uid; //唯一标识符
       private Page pg; //页面对象
   }
   ```

8. data()返回的是共享数组；before()修改数据项之前调用，用于保存原始数据oldRaw；unbefore()在撤销修改时调用，用于恢复原始数据；after()用于记录日志(添加/更新)并解锁数据项；release()释放DataItem缓存

9. 数据是raw的部分，而不是pg里面的data??

##### 5 DataManager

1. DataManager是DM层直接对外提供方法的类，同时页实现成DataItem对象的缓存。DataItem存储的key，是由页号和页内偏移组成的8字节无符号整数，页号和偏移各占4字节。
2. uid=pgno<<32|offset；通过运算能重新计算得到pgno和offset；getForCache()只需要从 key 中解析出页号，从 pageCache 中获取到页面，再根据偏移，解析出 DataItem 即可
3. DataManager初始化：创建有两种流程，1是从空文件创建，需要对第一页进行初始化；2是从已有文件创建，需要对第一页进行校验，来判断是否需要执行恢复流程，并重新对第一页生成随机字节
4. 从空文件创建create()；分别创建PageCache, Logger对象(TM对象)，再创建DataManagerImpl对象，初始化PageOne第一个界面
5. 从已有文件创建open()；分别打开PageCache, Logger对象(TM对象)，再创建DataManagerImpl对象，检查PageOne第一个界面，如果不符合规则就进行恢复操作；从第二页开始将每一页编号以及空闲空间大小添加到PageIndex；设置PageOne为打开状态
6. DM提供三个功能：read(), insert(), update()，修改是通过DataItem实现
7. read()：根据UID从缓存中获取的DataItem，并校验有效位；  insert()：再PageIndex中获取一个足以存储插入内容的页面的页号，获取页面后，首先需要写入插入日志，接着才可以通过PageX插入数据，并返回插入位置的偏移，最后需要将页面信息重新插入pageIndex
8. 资源的UID就是根据所存储的pgno和offset确定的，pgno也对应内存中的位置，直接乘pageSize就得到磁盘文件中的内存位置



1. 对于insert()：1先把数据包装成DataItem(ValidFlag, size, data)，然后从pIndex中挑选合适空余空间的pageNo，然后page直接通过PageCache读取(根据pageNo读page对应的那块内存的全部内容)，insert到page.data中，再生成pageNo和offset生成uid；近似效果等于cache.put(uid, di)，此后就能够通过uid读取到di
2. 对于read(uid)：DataManager继承缓存框架，根据uid直接用pageCache(DataManager直接包括了pc)查找到page，然后包装为DataItem类，raw=pg.data；近似效果就是从cache Map<uid, di>根据uid查找di；后面根据读取到的data，可以使用data()共享内存，before()拷贝old_raw，after()记录日志
3. 所以插入读取的功能其实就是通过uid或者pageNo的，确定了pageNo和offset直接读page.data就是了，page.data就是DataItem(ValidFlag, size, data)的格式
4. pageCache和DataManager都继承了缓存框架，pageCache存的是page的缓存，DataManager存的是dataItem的缓存(包含了page)
5. PageCache是.db文件，logger是.log文件，page是哪个文件？
6. DM中事务管理体现在哪？(—>在VM中abort()手动调用或者在检测出死锁时会回滚，或者说TM就是供VM使用的)目前是记录log时会附带xid，难道是在redo持久化和undo撤销时才开启事务吗？
7. 所以需要搞清楚上层TBM是怎么获取uid的?
8. undo log 和 redo log在什么时候开启
9. release释放的都是缓存框架里的缓存，也就是cache Map，同时涉及引用计数法，所以从cache获取的page和dataItem要及时release



##### 6VM 记录的版本与事务隔离

1. VM基于两段锁实现了调度序列的可串行化，并实现MVCC以消除读写阻塞，同时实现两种隔离级别

2. MYDB采用两段锁协议(2PL)来实现，保证了调度序列的可串行化，但是不可避免地导致了事务间的相互阻塞，甚至死锁。为了提高事务处理的效率，降低阻塞概率，实现了MVCC

3. DM层向上层提供了数据项(DataItem)的概念，VM通过管理所有的数据项，向上层提供记录(Entry)的概念。上层模块通过VM操作数据的最小单位，就是记录。VM在内部为每个Entry维护了多个版本。

4. Entry的格式：  class包含uid, dataItem, vm

   > **[XMIN]	[XMAX]	[DATA]**

   1. **XMIN** 是创建该条记录（版本）的事务编号
   2. **XMAX **则是删除该条记录（版本）的事务编号
   3. **DATA **就是这条记录持有的数据

5. data()方法调用dataItem.data()再拼接xmin, xmax；setXmax()方法修改数据时，先后调用dataItem的before()方法保存旧值；data()方法；after()生成修改日志

6. 读提交的事务可见性逻辑  boolean readCommited(判断是否可见 

   ```
   (XMIN == Ti and                             // 由Ti创建且
       XMAX == NULL                            // 还未被删除
   )
   or                                          // 或
   (XMIN is commited and                       // 由一个已提交的事务创建且
       (XMAX == NULL or                        // 尚未删除或
       (XMAX != Ti and XMAX is not commited)   // 由一个未提交的事务删除
   ))
   ```

7. 可重复读的可见性逻辑  boolean repeatableRead()判断是否可见

   ```
   (XMIN == Ti and                 // 由Ti创建且
    (XMAX == NULL or               // 尚未被删除
   ))
   or                              // 或
   (XMIN is commited and           // 由一个已提交的事务创建且
    XMIN < XID and                 // 这个事务小于Ti且
    XMIN is not in SP(Ti) and      // 这个事务在Ti开始前提交且
    (XMAX == NULL or               // 尚未被删除或
     (XMAX != Ti and               // 由其他事务删除但是
      (XMAX is not commited or     // 这个事务尚未提交或
   XMAX > Ti or                    // 这个事务在Ti开始之后才开始或
   XMAX is in SP(Ti)               // 这个事务在Ti开始前还未提交
   ))))
   ```

   所以需要保存事务创建时活跃事务id，Map<Long, Boolean>

8. 

##### 7VM 死锁检测与VM的实现

1. MVCC可能导致版本跳跃问题，MYDB如何避免2PL导致的死锁

2. 版本跳跃问题是指在多版本并发控制（MVCC）中，一个事务要修改某个数据项时，可能会出现跳过中间版本直接修改最新版本的情况，从而产生逻辑上的错误。解决版本跳跃的思路：如果Ti需要修改X，而X已经被Ti不可见的事务Tj修改了，那么要求Ti回滚；

3. 版本跳跃的检查：取出要修改的数据 X 的最新提交版本，并检查该最新版本的创建者对当前事务是否可见；  读提交允许版本跳跃，可重复读不允许；     isVersionSkip()

4. 在基于2PL两阶段锁协议的并发控制中，事务A等待另一个事务B释放锁的等待关系可以被抽象成有向边，构成等待图。检测死锁只需要查看等待图中是否存在环即可

5. LockTable：维护一个等待图，以进行死锁检测，维护5个Map，记录事务ID(XID)和资源ID(UID)之间的持有或者等待的关系

   ```java
   public class LockTable {
       // 某个XID已经获得的资源的UID列表，键是事务ID，值是该事物持有的资源ID列表。
       private Map<Long, List<Long>> x2u;
       // UID被某个XID持有,键是资源ID，值是持有该资源的事务ID。
       private Map<Long, Long> u2x;
       // 正在等待UID的XID列表，键是资源ID，值是正在等待该资源的事务ID。
       private Map<Long, List<Long>> wait; 
       // 正在等待资源的XID的锁,键是事务ID，值是该事务的锁对象。
       private Map<Long, Lock> waitLock;   
       // XID正在等待的UID,键是事务ID，值是该事务正在等待的资源ID。
       private Map<Long, Long> waitU;
       // 一个全局锁，用于同步。
       private Lock lock;
   }
   ```

6. add()在每次出现等待的情况时，就尝试向图中添加一条边，并进行死锁检测。如果检测到死锁，就撤销这条边，不允许添加，并撤销该事务

7. 查找是否有环的算法：利用dfs

   ```java
   流程：初始化xidStamp=new HashMap(){xid, stamp}, stamp=2，dfs(xid=1)
   
   第一遍：xidStamp = null，stamp=2
   private boolean dfs(long xid) { //xid = 1
       Integer stp = xidStamp.get(xid); // null
       xidStamp.put(xid, stamp); // 1,2
   
       Long uid = waitU.get(xid); // uid = 2
       Long x = u2x.get(uid); // x = 2
   
       return dfs(x); // 将2存入进去
   }
   
   1. xidStamp={(1,2)}；事务1在等待资源2，资源2被事务2持有，dfs(2)
   2. xidStamp={(1,2),(2,2)}；事务2在等待资源3，资源3被事务3持有，dfs(3)
   3. xidStamp={(1,2),(2,2),(3,2)}；事务3在等待资源1，资源1被事务1持有，dfs(1)
   4. 发现事务1已经包含了时间戳2，且等于当前时间戳2，证明存在死锁
   5. (如果发现包含的是时间戳1，小于当前时间戳2，说明这事务id已经被检查过了且没有发现死锁)
   ```

8. remove()，当一个事务commit或者abort时，就会释放掉它自己持有的锁，并将自身从等待图中删除；还需要移除该事务占用的所有资源，并从等待队列中挑选新的事务来占用该资源

9. 

##### 7VM的实现

1. ```java
   public interface VersionManager {
       byte[] read(long xid, long uid) throws Exception;
       long insert(long xid, byte[] data) throws Exception;
       boolean delete(long xid, long uid) throws Exception;
   
       long begin(int level);
       void commit(long xid) throws Exception;
       void abort(long xid);
       
   }
   //具体的实现即实现上述6个方法
```
   
2. VM的实现类还被设计为Entry的缓存，需要继承‘AbstractCache\<Entry>’

3. begin()：新建事务，调用tm begin方法，如果隔离类别为0则创建快照保存当前活跃事务id

4. commit()：移除事务，释放该事务的锁

5. abort()：除了手动调用abort方法，在事务被检测出死锁或者出现版本跳跃时，也会自动回滚；也是移除事务(lockTable)和释放锁

6. read()：判断可见性，返回data

7. insert()：将数据包裹成Entry，然后交给DM插入

8. delete()：先获取entry，在判断数据项是否被当前事务删除、版本是否被跳过，没有则将数据项的xmax设置为当前事务ID，标识数据项被删除

9. 

##### 8IM 索引管理

1. 基于B+树的索引，IM直接基于DM，索引的数据被直接插入数据库文件中，而不需要经过版本管理

2. 基本结构

   ```
   [LeafFlag][KeyNumber][SiblingUid]
   [Son0][Key0][Son1][Key1]...[SonN][KeyN]
   ```

   - **[LeafFlag]**：标记该节点是否为叶子节点
   - **[KeyNumber]**：该节点中 key 的个数
   - **[SiblingUid]**：是其兄弟节点存储在 DM 中的 UID，用于实现节点的连接
   - [**SonN] [KeyN]**：后续穿插的子节点，最后一个 Key 始终为 MAX_VALUE，以方便查找

3. SonN和KeyN分别是children结点和键值，并且是一一对应的；键值就是创建索引根据的数据值，真实储存的数据应该就是uid，通过uid能找到所有数据

4. Node类持有了其B+树结构的引用，DataItem的引用和SubArray的引用，用于方便快速修改和释放数据

5. 

6. IM层对上层模块主要提供两种能力：插入索引和搜索节点。

7. 此外磁盘上存取的数据为二进制数组，转为对象时可以直接就保存为字节数组，后面再结果解析转化为明确意义的属性值

8. 

##### 9TBM 字段与表管理

1. TBM实现了对字段结构和表结构的管理
2. SQL解析器：Parser实现了对类SQL语句的结构化解析，将语句中包含的信息封装为对应语句的类。
3. parse包的**Tokenizer**类，对语句进行逐字节解析，根据空白符或者上述词法规则，将语句切割成多个token。对外提供了peek(), pop()方法方便取出Token解析解析。
4. **Parser 类**则直接对外提供了 `Parse(byte[] statement)` 方法，核心就是一个调用 Tokenizer 类分割 Token，并根据词法规则包装成具体的 Statement 类并返回。解析过程很简单，仅仅是根据第一个 Token 来区分语句类型，并分别处理，不再赘述。



1. 字段与表管理：管理表和字段的数据结构，如表名，表字段信息和字段索引等
2. 数据存储结构：表和字段的信息以二进制形式存储在数据库的 Entry 中。
3. **字段信息表示：** 字段的二进制表示包含字段名（FieldName）、字段类型（TypeName）和索引UID（IndexUid）。
   - 字段名和字段类型以及其他信息都以字节形式的字符串存储。
   - `**[FieldName] [TypeName] [IndexUid]**`
   - 为了明确字符串的存储边界，采用了一种规定的字符串存储方式，即在字符串数据之前存储了字符串的长度信息。
   - `**[StringLength] [StringData]**`
4. **字段类型限定：** 字段的类型被限定为 int32、int64 和 string 类型。
5. **索引表示：** 如果字段被索引，则IndexUid指向了索引二叉树的根节点；否则该字段的IndexUid为0。
6. **读取和解析：** 通过唯一标识符（UID）从虚拟内存（VM）中读取字段信息，并根据上述结构解析该信息。







把 /tmp/mydb 替换为 D:/MYDB/mydb





























