## CountDownLatch是什么
> CountDownLatch可以控制线程的执行，他可以让所有持有他的多个线程同时执行，也可以控制单个线程执行。
> 
> 他初始化的时候会传出一个int类型的参数i，调用一次countDown(）方法后i的值会减1。
> 
> 在一个线程中如果调用了await()方法，这个线程就会进入到等待的状态，当参数i为0的时候这个线程才继续执行。

## 跑步比赛demo

**例子：**

5个选手参加比赛，第一，必须满足五个选手都准备就绪才宣布比赛开始；第二，当五个选手都完成跑步，比赛才结束。

**思路：**

五个选择都是独立的线程，每个线程都持有开始比赛的触发点，即都持有同一个CountDownLatch。
主线程也要持有结束比赛的触发点CountDownLatch，当每个选手跑步完成，执行一次countDown。

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CountDownLatchTest {

    private static final int PLAYER_COUNT = 5;

    public static void main(String[] args) {
        CountDownLatch playStart = new CountDownLatch(1);
        CountDownLatch playEnd = new CountDownLatch(PLAYER_COUNT);

        Player[] players = new Player[PLAYER_COUNT];
        for (int i = 0; i < PLAYER_COUNT; i++) {
            players[i] = new Player("player" + i, playStart, playEnd);
        }

        System.out.println("进入准备阶段===================================");

        ExecutorService executorService = Executors.newFixedThreadPool(PLAYER_COUNT);
        for (int i = 0; i < PLAYER_COUNT; i++) {
            executorService.execute(players[i]);
        }

        try {
            Thread.sleep(1000);//设置一个准备时间阶段，等待所有选手准备完成
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("完成准备阶段===================================");
        playStart.countDown();//触发统一的开跑时间点
        System.out.println("比赛开始================================");

        try {
            playEnd.await();//等待所有选手完成跑步
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println("比赛结束=================================");
            executorService.shutdown();//关闭线程池
        }
    }

    private static class Player implements Runnable {
        private String name;
        private CountDownLatch start, end;

        public Player(String name, CountDownLatch start, CountDownLatch end) {
            this.name = name;
            this.start = start;
            this.end = end;
        }

        @Override
        public void run() {
            System.out.println(name + "准备完成");
            try {
                start.await();//等待start.countDown，触发开始
                int time = (int) (Math.random() * 10000);
                System.out.println(name + "开跑, 需要" + time);
                Thread.sleep(time);//设置当前选手跑步的时间
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.println(name + "完成");
                end.countDown();//当前选手完成跑步
            }
        }
    }
}
```

运行结果：
> 进入准备阶段===================================
> 
> player0准备完成
> 
> player1准备完成
> 
> player2准备完成
> 
> player3准备完成
> 
> player4准备完成
> 
> 完成准备阶段===================================
> 
> 比赛开始================================
> 
> player0开跑, 需要6525
> 
> player4开跑, 需要9459
> 
> player3开跑, 需要3111
> 
> player2开跑, 需要857
> 
> player1开跑, 需要8808
> 
> player2完成
> 
> player3完成
> 
> player0完成
> 
> player1完成
> 
> player4完成
> 
> 比赛结束=================================