---
layout: post
title: "JUC核心之手写mini版ReentrantLock"
date: 2021-05-06
description: "JUC核心之手写mini版ReentrantLock"
tag: Java基础


---



直接进入主题，看代码。

```java
/**
 * 加锁与释放锁的接口
 * 子类实现各自加锁释放锁的功能
 * @Author Jay Chen
 * @Date 2021/5/6 2:14 下午
 * @Version 1.0
 */
public interface MiniLock {

    void lock();
    void unLock();
}
```



```java
import sun.misc.Unsafe;
import java.lang.reflect.Field;
import java.util.concurrent.locks.LockSupport;

/**
 * @Author Jay Chen
 * @Date 2021/5/6 2:16 下午
 * @Version 1.0
 */
public class MiniReentrantLock implements MiniLock {

    /**
     * 通过state表示锁标志，锁的是什么？？？
     * 锁的资源
     * 0：表示无锁状态
     * 大于0：则表示锁被占用
     */
    private volatile int state;

    /**
     * 持有锁线程(独占锁模式)
     */
    private Thread exclusiveOwnerThread;

    /**
     * 需要两个节点引用维护阻塞队列
     * 头节点：head;head占用的线程就是当前线程exclusiveOwnerThread
     * 尾节点：tail
     */
    private Node head;
    private Node tail;

    /**
     * 设置头节点
     *
     * @param node node
     */
    private void setHead(Node node) {
        // 设置为头节点
        this.head = node;
        // 将node封装的线程设置为null
        // why？？？
        // 因为此node已经获取到锁了。
        node.thread = null;
        // 帮助上一个头节点出队列，断开其连接关系。
        node.prev = null;
    }

    /**
     * 存放阻塞的线程
     * Node节点，FIFO队列
     */
    static final class Node {
        /**
         * 封装的线程本身
         */
        Thread thread;

        /**
         * 前置节点引用
         */
        Node prev;

        /**
         * 后置节点引用
         */
        Node next;

        public Node(Thread thread) {
            this.thread = thread;
        }

        public Node() {
        }

        public Thread getThread() {
            return thread;
        }

        public void setThread(Thread thread) {
            this.thread = thread;
        }

        public Node getPrev() {
            return prev;
        }

        public void setPrev(Node prev) {
            this.prev = prev;
        }

        public Node getNext() {
            return next;
        }

        public void setNext(Node next) {
            this.next = next;
        }
    }

    /**
     * 获取锁，获取不到锁则阻塞park
     * 模拟的是公平锁
     * 先来分析一下lock的过程
     * 场景1：线程进来后发现stats == 0；则直接尝试抢占锁。
     * 场景2：线程进来后发现stats > 0
     * 1:如果当前持有锁的线程是当前线程则重入锁，state + 1
     * 2:如果当前持有锁的线程不是当前线程，则当前线程则需要进入队列
     */
    @Override
    public void lock() {
        this.acquire(1);
    }

    /**
     * 竞争资源
     * 1：尝试获取锁资源
     * 2：抢占锁失败，则阻塞当前线程（加入等待队列）。
     * 3：竞争锁资源
     *
     * @param arg
     */
    private void acquire(int arg) {
        if (!tryAcquire(arg)) {
            // 尝试加锁失败
            Node node = addWaiter();
            this.acquireQueued(node, arg);
        }
    }

    private void acquireQueued(Node node, int arg) {
        // 只有当前线程获取到锁才会退出自旋
        for (; ; ) {
            // 只有一种情况才能有资格去抢占锁，只有当node节点为head的后继节点
            Node prev = node.getPrev();
            // prev == this.head：判断当前节点前继节点是否为head节点
            if (prev == this.head && this.tryAcquire(arg)) {
                // 进入到这里则说明抢占到锁了
                // 1：将head节点设置为当前节点
                this.setHead(node);
                // 2：协助old head节点出队列,将old head节点与后继节点断开引用关系。
                prev.next = null; // help gc
            }
            System.out.println("线程挂起：" + Thread.currentThread());
            LockSupport.park();
            System.out.println("线程唤醒：" + Thread.currentThread());
        }
    }

    /***
     * 尝试抢占锁失败？？？做哪些事情？？？
     * 1：需要将当前线程封装成Node节点，加入到阻塞队列
     * 2：需要将当前线程 park掉，使线程处于挂起状态
     *
     * 唤醒之后做什么事情？？？
     * 1：检查当前节点node是否为 head.next节点，因为只有head.next节点才有资格去抢占锁资源
     * 2：抢占
     *    成功：将当前node设置为head节点，帮助old head出队列，设置当前持有锁线程为当前线程，返回业务层面
     *    失败：继续park，等待被唤醒......
     *
     */
    private Node addWaiter() {
        Node newNode = new Node(Thread.currentThread());
        // 接下来就是入队列逻辑
        // 1：找到newNode.prev
        // 2：更新newNode.prev
        // 3：cas更新tail节点为newNode节点
        // 4：更新prev.next为newNode节点
        Node prev = tail;
        if (prev != null) {
            newNode.prev = prev;
            // CAS更新tail节点
            if (this.compareAndSetTail(prev, newNode)) {
                prev.next = newNode;
                return newNode;
            }
        }

        // 执行到这里有两种情况
        // 1：队列为空队列
        // 2：加入队列失败
        this.enq(newNode);
        return newNode;
    }

    /**
     * 自旋入队列，只有成功后才会返回
     * 1： tail == null 表示当前队列为空
     * 2： cas设置当前newNode 为 tail节点时失败。其他线程抢先一步
     */
    public void enq(Node node) {
        for (; ; ) {
            // 第一种情况
            // tail == null 表示当前队列为空
            // 白话来讲就是当前线程是第一个抢占锁失败的。
            // 当前持有锁的线程，我们之前并没有设置任何node节点，所以需要将当前持有锁的线程封装为一个head节点
            // 随后将当前node设置为持有锁的node节点的第一个后继节点。
            if (tail == null) {
                // CAS设置head节点
                if (this.compareAndSetHead(new Node())) {
                    tail = head;
                    // 注意这里并没有退出自旋。
                    // 还会自旋，这里只是设置了一个head节点。后面的自旋才是将当前节点加入阻塞队列
                }
            } else {
                Node prev = tail;
                if (prev != null) {
                    node.prev = prev;
                    // CAS更新tail节点
                    if (this.compareAndSetTail(prev, node)) {
                        prev.next = node;
                        // 退出自旋
                        return;
                    }
                }
            }
        }
    }


    /**
     * 尝试获取锁
     *
     * @param arg
     * @return true:成功，false:失败
     */
    private boolean tryAcquire(int arg) {
        // 判断当前锁标志状态是否==0
        if (state == 0) {
            // 当前stats == 0时，并不一定能抢到锁。
            // 因为模拟的是公平锁，则需要判断前面是否有线程在等待锁。得讲究先来后到
            // this.hasQueuedPredecessor():判断是否有资格抢占锁资源
            // compareAndSetState：cas更改state值，这里为什么使用CAS操作？？？因为lock模拟的是多线程
            if (!this.hasQueuedPredecessor() && compareAndSetState(0, arg)) {
                // 抢占锁成功了需要做什么？？
                // 设置当前持有锁的线程为当前线程
                this.exclusiveOwnerThread = Thread.currentThread();
                return true;
            }

        } else if (this.exclusiveOwnerThread == Thread.currentThread()) {
            // 这里重入锁
            int c = getState();
            c = c + arg;
            // 思考这里为什么不需要用CAS进行操作？？？
            // 因为这里是自身已经持有锁的状态，所以不需要。
            this.state = c;
            return true;
        }
        // 1：CAS加锁失败
        // 2：当前state > 0 且 当前持有锁的线程不是当前线程
        return false;
    }

    /**
     * 判断当前是否有线程在等待线程
     * 总结：什么情况下才会返回false
     * 1：阻塞队列还未初始化
     * 2：阻塞队列已经初始化，但是head == tail，表示还未有线程等待锁
     * 3：head.next 节点线程 = Thread.currentThread()
     *
     * @return true:表示当前线程前面有线程在等待锁资源，false：表示没有。
     */
    private boolean hasQueuedPredecessor() {
        Node h = head;
        Node t = tail;
        Node s;
        // h != t
        // 成立：则说明已经初始化阻塞队列，并且已经有node节点在等待锁了
        // 不成立：1）h = t == null; 表示还未有线程在等待锁。
        //        2）h = t == head 第一个线程获取锁失败，会为当前持有锁线程 补充创建一个node节点。

        // (s = h.next) != null || s.getThread() != Thread.currentThread()

        // (s = h.next) != null
        // 为什么会进行(s = h.next) != null这样的判断？？？
        // 多线程情况下，有一种极端的情况，第一个获取锁失败的线程会为持有锁的线程补充创建一个node节点。随后会继续自旋，将自己添加进阻塞队列中。
        // 接下来的自旋会有一个步骤CAS 设置tail节点（逻辑在enq(newNode)中），注意此时还未设置head.next节点;

        // s.getThread() != Thread.currentThread()
        // 判断头节点的后继节点的线程是否等于当前线程
        return h != t && ((s = h.next) != null || s.getThread() != Thread.currentThread());
    }

    /**
     * 释放锁
     */
    @Override
    public void unLock() {
        release(1);
    }

    private void release(int arg) {
        // 如果条件成立，则说明锁已经被完全释放，那么就需要唤醒后面等待的线程
        if (this.tryRelease(arg)) {
            Node h = this.head;
            if (h.next != null) {
                this.unParkSuccessor(h);
            }
        }
    }

    /**
     * 唤醒后继线程
     *
     * @param node
     */
    private void unParkSuccessor(Node node) {
        Node n = node.next;
        if (n != null && n.thread != null) {
            LockSupport.unpark(n.thread);
        }
    }

    /**
     * 尝试释放锁
     *
     * @param arg
     * @return true：表示锁已经完全释放
     */
    private boolean tryRelease(int arg) {
        int c = this.getState() - arg;
        // 判断当前释放锁的线程是否等于持有锁的线程
        if (this.getExclusiveOwnerThread() != Thread.currentThread()) {
            throw new Error("must get lock");
        }

        // 条件成立表示锁已经完全被释放
        if (c == 0) {
            // 这里需要做两件事情。
            // 1：将ExclusiveOwnerThread置为null
            // 2：将state置为0
            this.exclusiveOwnerThread = null;
            this.state = c;
            return true;
        }
        return false;
    }
		
  	/**
  	* CAS修改相关字段
  	*/
    private static final Unsafe unsafe;
    private static final Long stateOffset;
    private static final Long headOffset;
    private static final Long tailOffset;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
            stateOffset = unsafe.objectFieldOffset(MiniReentrantLock.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset(MiniReentrantLock.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset(MiniReentrantLock.class.getDeclaredField("tail"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

    public int getState() {
        return state;
    }

    public Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }

    /**
     * cas修改head
     *
     * @param update
     * @return
     */
    private final boolean compareAndSetHead(Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }

    /**
     * cas 修改tail
     *
     * @param expect
     * @param update
     * @return
     */
    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }

    /**
     * cas 修改state
     */
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
}

```



如果这代码能看懂，那看源码ReentrantLock应该不难。

