##### 第一题

```
package naverchina;

import java.util.ArrayList;
import java.util.List;

/**
 * @author boykait
 * @version 1.0
 * @date 2020/3/19 21:49
 */
class Node<V> {
    /**
     * 节点值
     */
    private V value;
    /**
     * 是否被访问
     */
    private boolean visited;
    /**
     * 前置节点列表
     */
    private List<Node<V>> pres;
    /**
     * 后置节点列表
     */
    private List<Node<V>> nexts;

    public Node(V v) {
        this.value = v;
        this.pres = new ArrayList<>();
        this.nexts = new ArrayList<>();
    }

    public void addPreNode(Node<V> pre) {
        this.pres.add(pre);
    }

    public void addNextNode(Node<V> next) {
        this.nexts.add(next);
    }

    public V getValue() {
        return value;
    }

    public void setValue(V value) {
        this.value = value;
    }

    public List<Node<V>> getPres() {
        return pres;
    }

    public void setPres(List<Node<V>> pres) {
        this.pres = pres;
    }

    public List<Node<V>> getNexts() {
        return nexts;
    }

    public void setNexts(List<Node<V>> nexts) {
        this.nexts = nexts;
    }

    public boolean isVisited() {
        return visited;
    }

    public void setVisited(boolean visited) {
        this.visited = visited;
    }


}

public class Test1 {
    public static void main(String[] args) {
        List<Node<String>> nodes = initNodes();
        nodes.forEach(node -> {
            if (null == node.getPres() || node.getPres().size() == 0) {
                dfs(node);
            }
        });
    }

    /**
     * dfs遍历
     *
     * @param node 不含前置及节点列表的节点
     */
    private static void dfs(Node<String> node) {
        if (judge(node)) {
            System.out.println(node.getValue());
            node.setVisited(true);
        }
        if (null != node.getNexts() && node.getNexts().size() > 0) {
            node.getNexts().forEach(Test1::dfs);
        }
    }

    /**
     * 判断是否可以被访问
     *
     * @param node 当前节点 当前节点未访问 且所有的pres节点被访问
     * @return 判断结果
     */

    private static boolean judge(Node<String> node) {
        if (node.isVisited()) {
            return false;
        }
        if (null != node.getPres() && node.getPres().size() > 0) {
            List<Node<String>> pres = node.getPres();
            for (int i = 0; i < pres.size(); i++) {
                if (!pres.get(i).isVisited()) {
                    return false;
                }
            }
        }
        return true;
    }

    private static List<Node<String>> initNodes() {
        Node<String> nodeA = new Node<>("A");
        Node<String> nodeB = new Node<>("B");
        Node<String> nodeC = new Node<>("C");
        Node<String> nodeD = new Node<>("D");
        Node<String> nodeE = new Node<>("E");
        Node<String> nodeF = new Node<>("F");
        Node<String> nodeG = new Node<>("G");
        Node<String> nodeH = new Node<>("H");
        nodeA.addPreNode(nodeG);
        nodeA.addPreNode(nodeC);
        nodeA.addNextNode(nodeD);
        nodeB.addNextNode(nodeG);
        nodeB.addNextNode(nodeE);
        nodeC.addPreNode(nodeH);
        nodeC.addNextNode(nodeA);
        nodeD.addPreNode(nodeA);
        nodeD.addNextNode(nodeF);
        nodeE.addPreNode(nodeB);
        nodeE.addPreNode(nodeG);
        nodeE.addNextNode(nodeF);
        nodeF.addPreNode(nodeD);
        nodeF.addPreNode(nodeE);
        nodeG.addPreNode(nodeB);
        nodeG.addNextNode(nodeA);
        nodeG.addNextNode(nodeE);
        nodeH.addNextNode(nodeC);
        List<Node<String>> nodes = new ArrayList<>();
        nodes.add(nodeA);
        nodes.add(nodeB);
        nodes.add(nodeC);
        nodes.add(nodeD);
        nodes.add(nodeE);
        nodes.add(nodeF);
        nodes.add(nodeG);
        nodes.add(nodeH);
        return nodes;

    }
}

```

##### 第二题

```
package naverchina;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * @author boykait
 * @version 1.0
 * @date 2020/3/19 21:02
 */
class Cache<K, V> {
    private LinkedHashMap<K, V> caches;
    private static ReentrantReadWriteLock readWriteLock;
    private static Integer CAPABILITY = 0;

    public Cache(int capability) {
        readWriteLock = new ReentrantReadWriteLock();
        CAPABILITY = capability;
        caches = new LinkedHashMap<K, V>(capability, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > CAPABILITY;

            }
        };
    }

    public V get(K key) {
        ReentrantReadWriteLock.ReadLock readLock = readWriteLock.readLock();
        V v;
        readLock.lock();
        v = caches.getOrDefault(key, null);
        readLock.unlock();
        return v;
    }

    public boolean put(K key, V value) {
        try {
            ReentrantReadWriteLock.WriteLock writeLock = readWriteLock.writeLock();
            writeLock.lock();
            caches.put(key, value);
            writeLock.unlock();
            return true;
        } catch (Exception e) {
            System.out.println(e);
            return false;
        }
    }

    public boolean remove(K key) {
        try {
            ReentrantReadWriteLock.WriteLock writeLock = readWriteLock.writeLock();
            writeLock.lock();
            caches.remove(key);
            writeLock.unlock();
            return true;
        } catch (Exception e) {
            System.out.println(e);
            return false;
        }
    }
}

public class Test2 {
    public static void main(String[] args) {
        Cache<Integer, Integer> cache = new Cache<>(2);
        cache.put(1, 1);
        cache.put(2, 2);
        System.out.println(cache.get(1)); // 输出1
        cache.put(3, 3);
        cache.put(4, 4);
        System.out.println(cache.get(2)); // 输出null
        System.out.println(cache.get(3)); // 输出 3
        cache.remove(4);
        System.out.println(cache.get(4));  // 输出 null
        System.out.println(cache.get(1)); // 输出 null
    }
}

```

#### 第三题

```
package naverchina;

import java.util.Stack;

/**
 * @author boykait
 * @version 1.0
 * @date 2020/3/19 21:30
 */
class Queue<T> {
    private Stack<T> data;
    private Stack<T> backData;

    public Queue() {
        data = new Stack<>();
        backData = new Stack<>();
    }
    public boolean push(T v) {
        while (!data.isEmpty()) {
            backData.push(data.pop());
        }
        data.push(v);
        while (!backData.isEmpty()) {
            data.push(backData.pop());
        }
        return true;
    }
    public T pop() {
        return !data.isEmpty() ? data.pop() : null;
    }

    public T peek() {
        return data.peek();
    }

    public boolean empty () {
        return data.isEmpty();
    }
}
public class Test3 {
    public static void main(String[] args) {
        Queue<Integer> queue = new Queue<>();
        queue.push(1);
        queue.push(2);
        queue.push(3);
        System.out.println(queue.empty());
        System.out.println(queue.pop());
        System.out.println(queue.peek());
        System.out.println(queue.pop());
        System.out.println(queue.pop());
    }
}

```
