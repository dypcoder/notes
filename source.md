HashMap:
	数据结构===>首先在外部看来是K-V形式的，其内部结构为K-Table形式，其中table中又包含node的数据结构，每一个节点与其上下关联的节点都是一样的结构，(这样的话就可以很好地利用递归思想来解决一些问题), 每个节点都是一个数组，类似于长了树叶的树(所谓的桶)?数组的作用是当发生hash冲突的
				时候用来存储其他的数据的，在1.8之前一直都是数组存储hash冲突的数据，1.8之后当数组容量超过8个之后，树叶转变为红黑树数据结构来提高效率。
	put     ===>              
	
	```java
	public V put(K key, V value) {
	    //这边有个为key计算hash值的操作,hash算法还不太明白。。。
        return putVal(hash(key), key, value, false, true);
    }

	//真正的put操作
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
		//定义一下下面可能用到的局部变量,防止操作的时候操作了原来存储的类变量，其实就是将原本对象的引用指向局部变量		   
        Node<K,V>[] tab; Node<K,V> p; int n, i;
		//判断是否为空及将成员变量的引用指向局部变量
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
		//这边将最后一个节点与hash值做位运算得到了下一个位置？其实这边是一个链表而不是数组？	
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }	
	```