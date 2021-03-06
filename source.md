### HashMap:


    数据结构:首先在外部看来是K-V形式的，其内部结构为K-Table形式，其中table中又包含node的数据结构，每一个节点与其上下关联的节点都是一样的结构，
    (这样的话就可以很好地利用递归思想来解决一些问题), 每个节点都是一个链表，类似于整齐排列的水桶，数组指定位置存取都比较快，当发生hash冲突的
	时候，数组下标所在的元素的开始转变为链表了，在1.8之前一直都是数组存储hash冲突的数据，1.8之后当数组容量超过8个之后，树叶转变为红黑树数据结构来提高效率。

#### put解析
***
```java
public V put(K key, V value) {
	    //这边有个为key计算hash值的操作,hash算法的作用主要是不同的key尽量算出不同的hash值
        return putVal(hash(key), key, value, false, true);
    }

    //真正的put操作
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
//定义一下下面可能用到的局部变量,防止操作的时候操作了原来存储的类变量，其实就是将原本对象的引用指向局部变量(注意这边的tab是一个Node数组) 其实操作的还是类变量
        Node<K,V>[] tab; Node<K,V> p; int n, i;
	//判断是否为空及将成员变量的引用指向局部变量
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
//这边利用hash方法处理之后的hash值(hash函数将高位也参与到运算中，高位称之为扰动函数参与hash值的运算，大大增加了散列型，1.7版本是经过4次位运算，1.8由于加入红黑树的情况只是做了这些处理)
//跟数组大小做按位与运算获得下标，如果这个数组下标处为null的话，直接放入一个新的node。
        if ((p = tab[i = (n - 1) & hash]) == null)｛
//调用一下node的构造方法向table中加入一个新元素，相当于新增列
            tab[i] = newNode(hash, key, value, null);
        //如果数组中该位置不为null，则node要进行纵向延伸，相当于table新增行
		}else {
	//定义两个局部变量
            Node<K,V> e; K k;
	//判断所存在node的hash值与key是否与要put的键值对的key相同，相同的话就将存在的node赋值给两个局部变量，然后执行覆盖值的操作(这边就是两个key相同的情况，value会被覆盖)
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
	//如果该链表为树，调用红黑树的put方法，这边看之后的红黑树put源码解读	
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
	//这边开始递归遍历node，直到找到为空的尾部节点然后将node放入链表的尾部
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
		//在这里遍历的时候会记数，当链表的数量超过默认值(8)的时候，将原本的节点转为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
		//由于有下面的p=e每次都会为p赋新值，这边的判断是为了判断链表中的后续节点中出现key相同的情况，key相同的话还是直接覆盖value的操作
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
		//为p赋新值,node的next节点	
                    p = e;
                }
            }
	//前面只是为了找存储的位置，位置找到就该存了
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
	//这里有个标记变量，如果为true的话key相同的情况下，新put的值将不会被替换
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
	//这个代码其实没什么用，这个方法是为了linkedHashMap服务的，在HashMap中没有作用	
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
	//判断是否需要扩容
        if (++size > threshold)
            resize();
	//这个同样也是服务于linkedHashMap的	
        afterNodeInsertion(evict);
        return null;
    }	
```
#### resize解析
***
```java
final Node<K,V>[] resize() {
    //新建一个引用指向原来的hashTable
        Node<K,V>[] oldTab = table;
        //如果new一个HashMap的话初始化操作并不在HashMap的构造函数中完成而是在
        //put的时候会触发resize(因为 ++size>threshold)完成hashTable的大小初始化
        //指定大小的话就是按照指定额大小初始化，没指定就是16
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //这边判断容量大小是否已经大于 1 << 30(即Int最大值除了最高位其他为都为0)
            //大于的话就让最大容量等于int最大值，不在扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //没到最大的话就将原先的容量扩大两倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //这就是上面所说的指定初始大小创建HashMap的勤快
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        //都没有就按默认大小初始化
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //同样是指定初始大小new HashMap的时候初始化一些值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        //新建一个HashTable开始向新表转移数据
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //循环遍历Node数组转移数据到新的HashTable
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                //新建局部变量引用原HashTable的每个数据
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    //将数组中的元素置为null便于垃圾回收
                    oldTab[j] = null;
                    //如果该处没有变为链表则直接放入新HashTable
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //如果该处元素是红黑树的话按红黑树处理
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //该处元素已经转化为链表，则需要对链表的next节点做处理
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        //上面定义了三个局部变量，用来转移原有链表到新HashTable
                        //下面就是循环链表所有节点并通过局部变量的引用最后将局部变量的值放入新hashTable
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
这里扩容后的下标位置计算很有意思，之前的做法是通过hash算法(即新put一组kv计算法方式一样)去计算在新数组中的位置的，1.8修改了这种方式，更加节省计算机资源，(n-1)&hash(key)通过按位与得到的结果来判断原有数组(长度为n)在新数组(长度2n)是否需要+n，如果结果是1则将原来的下标+n，否则不变，这样在保证分散性良好的情况下，相比之前节省了计算机资源。
