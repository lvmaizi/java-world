- hashmap底层原理是什么？
HashMap 数据结构是hash表也叫散列表，这是基础
1. 直接新建HashMap ,调用put方法，进入putVal，源码分析如下
```$xslt
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            //若hash表未初始化则对表进行初始化，默认数组长度为16，且扩容算法觉得长度一定为2的次方（后续resize()方法解释原因）
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            //由于数组长度n为2的次方，则（n-1）为 ****1111的结构，即二进制后几位均为1，这样就可以通过与运算
            //保证(n - 1) & hash结果既能够公平均匀分布到任意位置，又不会数组出界，且与运算效率高
            //若数组中对应位置为null,则新建节点放入
            tab[i] = newNode(hash, key, value, null);
        else {//桶节点不为null,即发生了hash冲突
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //key相等，则什么都不做，后续替换value
                e = p;
            else if (p instanceof TreeNode)
                //桶节点为红黑树节点，则将当前节点插入红黑树
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //桶节点为链表，死循环遍历列表，在尾部插入
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //尾部插入
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            //链表长度到达阈值8，链表转红黑树
                            treeifyBin(tab, hash);
                        break;
                    }
                    //key相等，替换value
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // 即上述所说替换value
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //适配LinkHashMap
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //若当前哈希表长度>阈值，则触发扩容，阈值根据负载因子计算
        if (++size > threshold)
            resize();
        //适配LinkHashMap
        afterNodeInsertion(evict);
        return null;
    }
```
2. 扩容方法分析，也包括初始化
```$xslt
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;//原始容量
        int oldThr = threshold;  //原始负载因子
        int newCap, newThr = 0;  //新的容量及负载因子
        if (oldCap > 0) { //扩容
            if (oldCap >= MAXIMUM_CAPACITY) {//已经到最大值，无法继续扩容
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&//容量扩大两倍
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // 默认  初始容量为16，阈值为12
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];//新建数组,将原有数组中的数据复制到新数组
        table = newTab;
        if (oldTab != null) {
            //原数组不为空，遍历复制
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)//没有hash冲突，即当前桶只有一个节点，直接复制到和原数组相同的对应位置
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)//若为红黑树，则进行拆分（后续TODO）
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // 有hash冲突，且不是红黑树，即为链表
                    //因为新数组长度增加，原有存值方式(n - 1) & hash 将会发生改变, 有两种可能
                    //举例 若原为(n - 1)为00001111 则新的（n-1）为00011111，可以发现由于扩容机制保证为2的次方且扩容2倍，则
                    //扩容后（n-1）与原有的只有一位bit不同，所有与运算结果也只有两种可能，自己画图理解
                        //一种可能：(与原有hash后位置相同的) 定义这种可能的头尾节点
                        Node<K,V> loHead = null, loTail = null;
                        //另一种：(与原有hash后位置不同的，且一定比原有位置大了原数组长度) 定义这种可能的头尾节点
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //这里可以等价转换为e.hash & (oldCap-1) == j
                            //相信若上述两种可能的分析理解了，也就很容易想到这，oldCap 为 000100000的结构
                            if ((e.hash & oldCap) == 0) {
                                //添加尾节点
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                //添加另一个链表的尾节点
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //将一种可能的节头点放入扩容后的数组中
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //将令一种可能的节头点放入扩容后的数组中
                        if (hiTail != null) {
                            hiTail.next = null;
                            //这里验证了上述说法（与原有hash后位置不同的，且一定比原有位置大了原数组长度）
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
3. 移除元素源码分析
```$xslt
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {//若通过hash判断出当前确定没有要移除的对象，则什么都不做
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //桶节点即为要移除的节点
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)//要移除的节点在红黑树中
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {//要移除的节点在链表中
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)//从红黑树中移除节点
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)//移除桶节点，
                    tab[index] = node.next;
                else//移除链表节点
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

## 后记
1. get containsValue 等方法不做分析，较简单
2. 红黑树相关方法略复杂，有时间单独列出讲解，但这些实现不影响我们对hashmap重要方法的理解
3. 当指定初始容量时，HashMap 还是会通过tableSizeFor方法将容量转换为最近的那个2的次方
3. 红黑树在移除元素时会在到阈值时退化成链表
