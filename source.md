<h3>HashMap:</h3>
	数据结构===>首先在外部看来是K-V形式的，其内部结构为K-Table形式，其中table中又包含node的数据结构，每一个节点与其上下关联的节点都是一样的结构，(这样的话就可以很好地利用递归思想来解决一些问题), 每个节点都是一个数组，类似于长了树叶的树(所谓的桶)?数组的作用是当发生hash冲突的
				时候用来存储其他的数据的，在1.8之前一直都是数组存储hash冲突的数据，1.8之后当数组容量超过8个之后，树叶转变为红黑树数据结构来提高效率。<br/>
	put     ===>  <br/>            
	***
	```Java
	public V put(K key, V value) {
	    //这边有个为key计算hash值的操作,hash算法还不太明白。。。
        return putVal(hash(key), key, value, false, true);
    }

    //真正的put操作
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
		//定义一下下面可能用到的局部变量,防止操作的时候操作了原来存储的类变量，其实就是将原本对象的引用指向局部变量(注意这边的tab是一个NOde数组)
        Node<K,V>[] tab; Node<K,V> p; int n, i;
		//判断是否为空及将成员变量的引用指向局部变量
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
		//这边利用hash方法处理之后的hash值(hash函数将高位也参与到运算中，高位称之为扰动函数参与hash值的运算，大大增加了散列型，1.7版本是经过4次位运算，1.8由于加入红黑树的清苦啊只是做了这些处理)
		//跟数组大小做按位与运算获得下标，如果这个数组下标处为null的话，直接放入一个新的node。
        if ((p = tab[i = (n - 1) & hash]) == null)
		//调用一下node的构造方法向table中加入一个新元素，相当于新增列
            tab[i] = newNode(hash, key, value, null);
        //如果数组中该位置不为null，则node要进行纵向延伸，相当于table新增行
		else {
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