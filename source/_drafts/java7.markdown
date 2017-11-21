---
layout: post
title: "Java并发编程实践笔记7：活性冒险"
---

# 活性冒险(Liveness Hazard)
我们对线程安全和活性的需求往往是矛盾的。我们使用锁来保证线程安全，但是大量的使用锁会导致死锁。相似地，我们使用线程池和信号量来控制资源消耗，但是对这一行为没有正确的认识往往会导致资源死锁。Java程序并不会自己从死锁状态恢复，我们需要格外注意我们的程序中是否有潜在的死锁。

# 死锁(Deadlock)
我们用一个简单的例子展示死锁，比如线程A，B，C对应三个锁a，b，c。线程A拿着锁a然后请求锁b，线程B拿着锁b请求锁c，线程C拿着锁c请求锁a。这就形成了一个死锁。显然，这是一个循环，无法打破，这时三个线程就卡在了这里，无法动弹。值得注意的是，**这样的死锁必须在特定的时序下才会发生**，这就体现了多线程环境下的复杂性。这种死锁在轻负载情况下可能并不会发生，但是在重负载下，发生的几率就急剧上升，这种隐患往往会很隐蔽。

上面这种死锁的原因是我们没有按照一定的顺序来请求锁，如果时序凑巧，就会发生死锁。但是解决方案也很简单：**对锁的请求要按照一定的顺序**。

下面是一种简单的情况：

``` java
public class LeftRightDeadlock {
    private final Object left = new Object();
    private final Object right = new Object();

    public void leftRight() {
        synchronized (left) {
            synchronized(right) {
                dosomething();
            }
        }
    }

    public void rightLeft() {
        synchronized (right) {
            synchronized (left) {
                dosomethingElse();
            }
        }
    }
}
```

上面这种情况一旦不凑巧就死锁了。但是上面这种情况还是很好辨认的，因为对`left`和`right`这两个锁的请求并没有按一定顺序。但是还有一种情况是不好发现的：

``` java
public void transferMoney(Account fromAccount,
            Account toAccount, DollarAmmount amount) 
                throws InsufficientFundsException {
    synchronized (fromAccount) {
        synchronized (toAccount) {
            if (fromAccount.getBalance().compareTo(amount) < 0) 
                throw new InsufficientFundsException();
            else {
                fromAccount.debit(amount);
                toAccount.credit(amount);
            }
        }
    }
}

```

上面这段代码看着很正常，实际上还是有死锁的。如果`fromAccount`和`toAccount`的参数一调换，我们实际上的两个锁的获取顺序也就变了，也就会出现死锁的可能。比较实用的解决方案是根据两个参数的哈希值大小来决定获取锁的顺序，万一哈希值一样，就使用一个额外的锁来确保同步。

但是上面的几段代码都是很好通过肉眼判断与分析的。一旦你的`synchronized`块中调用未了外部方法事情就复杂了，因为我们不知我们调用的方法的内部情况，也就出现了死锁的可能。

``` java 
class Taxi {
    private Point location, destination;
    private final Dispatcher dispatcher;

    public Taxi(Dispatcher dispatcher) { this.dispatcher = dispatcher; }
    public synchronized Point getLocation() { return location; }
    public synchronized void setLocation(Point location) {
        this.location = location;
        if (location.equals(destination))
            dispatcher.notifyAvailable(this);
    }
}

class Dispatcher {
    private final Set<Taxi> taxis;
    private final Set<Taxi> availableTaxis;

    public Dispatcher() {
        taxis = new HashSet<>();
        availableTaxis = new HashSet<>();
    }

    public synchronized void notifyAvailable(Taxi taxi) {
        availableTaxis.add(taxi);
    }

    public synchronized Image getImage() {
        Image image = new Image();
        for (Taxi t: taxis)
            image.drwaMarker(t.getLocation());
        return imagg;
    }
}
```

上面的`setLocation`和`getImage`实际上是都是获取了两个锁，但是顺序不同，这就是会造成死锁。

我们称调用一个没有持有任何锁的方法为`open call`。**在任何时候都要试着只使用open call，这会是你的程序更容易分析。**


# 如何避免死锁
尽量使用细粒度的锁，避免锁之间的交互。对于锁的请求要严格按照一定的顺序。

造成死锁的条件：

1.  互斥条件：一个资源只能在一段时间由一个线程占有
2.  请求与持有条件：一个线程在请求其他资源的时候依然持有自己手中的其他资源
3.  不可抢占条件：不能强行抢占一个线程持有的资源
4.  循环等待条件：资源的请求可能形成一个循环


