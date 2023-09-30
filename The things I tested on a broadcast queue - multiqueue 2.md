---  
title: The things I tested on a broadcast queue - multiqueue2  
updated: 2023-06-25 19:28:35Z  
created: 2023-06-25 19:28:07Z  
share: true  
---  
  
MultiQueue2 is a fast bounded mpmc queue that supports broadcast/broadcast style operations  
  
MultiQueue was developed by Sam Schetterer, but not updated for some time. I found it very useful as it implements futures. However, it is with a few outdated library API and the use of spin locks is taking 100% CPU in many cases.  
  
# Stone Age  
For any channels, if there is a potential for conflicts (more than one of the writers/readers are consuming the resources), there must be locks. So the 100% issue is very likely to be caused by a spinlock.  
  
At first, I did it in the `sleep way`  
  
https://github.com/abbychau/multiqueue2/blob/a850ed674ccb7c2d47fddcbe72070cca0efa6cc2/src/wait.rs#L31-L37  
  
Nonetheless, I sacrified the throughput because I punished every check.  
  
And then, I tried a much [smaller sleep](https://github.com/abbychau/multiqueue2/commit/67ff0f3dc8e33467125a7fe6b5a57b25747678eb). This is just meaningless because it does not trigger CPU from P-State to S-State. And I immediately find that it may be fool to cocerce S-State in a spinlock design.  
  
And then I tried 1ms,10ms,1000ns, for both settings of `check after wait` or `check before wait`. [link](https://github.com/abbychau/multiqueue2/commit/d3c9403762a34bbe11642419f66658844ec4d75d). It does not make much different.  
  
I tried the `_mm_pause` from `SSE2` as well, but it does not make any different.   
  
https://github.com/abbychau/multiqueue2/blob/d82603a08a4d4e0875054e7dd9ac0a63dee928d5/src/wait.rs#L31-L48  
  
Although `_mm_pause` is designed for spinlock, but the point is CPU will not switch P-State and S-State just for a few nono-seconds. It is simply not the right way.  
  
# Tool Age  
Keep trying the trival ways will never proceed more (although it practically solved the system problem of 100% cpu usage in my program and my program is not required to be very performant too), so I stepped back and rethought about my first solution.  
  
The first solution was actually doing less checking and forcing the cpu to sleep, so that it does not waste too much energy to do useless checkings which are 99%(roughly) predictable.  
  
What I really want to punish are the collided conflicts, because they are very likely to be blocked again if the CPU checks in the next cycle. At this time, sleeping is better than checking.  
  
So I make the spinlock to be a swing-back one.  
  
https://github.com/abbychau/multiqueue2/blob/74dd8254c6c6f8f4148ef2be9d4f8ed9d89e124d/src/wait.rs#L31-L42  
  
This works exactly as expected. The cpu usage lowered and the through-put is able to pass all the tests.  
  
# Futures  
However, I then found that the CPU usage is high for future queues. That, there is a kind of hybrid lock used for future queues.   
  
Therefore, I removed all the Spin before going to `parking_lot::Mutex::new(VecDeque::new())`. [link](https://github.com/abbychau/multiqueue2/commit/f311c7c02c392a656df85d662f8b3c1048536457)  
  
It increases the number of low level context switches very much but indeed lowed the CPU usage. By the nature of this heavy switch comparing to the light weight spinlock, the through-put of future queues becomes just 1/10 of the normal queue.  
  
And this experiment also taught me the performance ratio between `parking_lot` mutexes and native spins.  
  
# No Silver Bullet  
I really would like to come up with an equation for people to set the `try_spins` precisely, but it is very complicated because it includes all the envirnment information like CPU frequency, number of consumers, rate of feeding, etc.  
  
So I can just left it to the user.  
  
https://github.com/abbychau/multiqueue2/blob/master/src/multiqueue.rs#L1099-L1125  
  
I would like to investigate and develop a clear equation for concurrency hybrid lock, but it seems to be a bit long and I may have it later in a Paper form.  
  
Feel free to leave issues or comments.