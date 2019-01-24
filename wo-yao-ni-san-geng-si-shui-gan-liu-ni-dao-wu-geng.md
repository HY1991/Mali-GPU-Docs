---
description: kbase_backend_soft_hard_stop_slot
---

# 我要你三更死，谁敢留你到五更

当所有人都以为你要唠闲白的的时候，不唠也是一种闲白。

好吧，无闲白不技术，况且我知道有些人只看闲白。现在正是冬季，是吃羊的好时候，那么我们就讲讲（特别是给晚上看到这章的朋友们）吃羊。当然我相信我不是吃羊的行家，毕竟不是一个新内人，但是。。。相比较GPU驱动，我还是对吃羊更有信心。。。好吧。。。

羊肉大体可分为三种吃法：炖，烤，炒。依据我的经验，炒这种做法并不适合喜欢大块儿吃肉的朋友，比如我。那么就只剩下炖和烤了，其中烤这种做法能把羊肉的脂肪烤出来，洋溢的油脂让人胃口大开，明火会让羊肉产生美拉德反应，产生让人欲罢不能的风味。 ****但是吃不多，是的，即使在烤制的过程中有一部分油脂流失了，但保留的还是占绝大部分，更糟糕的是，变成液态的油脂更容易让人腻。

所以对于我个人来讲，我最钟意的还是炖羊肉，从咕嘟咕嘟的热锅里捞出一块儿羊肉，已经被煮的松散的肉被滑腻的胶原蛋白裹住。诶呀（这里我必须声明一下，这是东北话里的那个诶呀，那个“呀”发四声。。。），真好，不想写了，早睡觉明天吃肉去。。。

在Mali的驱动里提供了一个preempt的功能，就是说priority高的katom可以吧priority低的katom从JM和GPU里抢占掉。这一章主要介绍当确认要抢占的的时候，要进行怎样的操作，而不是如果决定是否抢占。

执行抢占的函数叫做**kbase\_backend\_soft\_hard\_stop\_slot**，他可以接受两种参数，一个是katom，一个是kctx。如果传入了katom，那么就会想办法停掉这个katom和它随后的katom （如果将要kill的katom是第0个，并且如果第1个和它属于相同的kctx，那么第1个katom也要被killed）。如果没有传入katom，而仅仅传入了一个kctx，那么就停掉这个slot上所有属于这个kctx的katom\(s\)。



如果idx0的rb\_state还没到SUBMITTED，在这种情况下，无论是idx0还是idx1都没有submitted到GPU上，直接跟去idx0\_valid和idx1\_valid来dequeue katoms并且返还给JS就好了。

如果idx0已经被submitted到GPU上，局势就变得没有那么明朗的，以至于要分类讨论。

1. 如果idx1也需要被停掉：

   1. 如果idx1也被提交到了GPU上：这个时候就需要读取JS\_COMMAND\_NEXT这个寄存器
      1. 如果寄存器里的值是0，那么就证明idx0已经完成了，也就是说idx1已经开始跑了，那么就需要STOP这个idx1
      2. 如果不是0，也就是说idx0还没有完成，就给JS\_COMMAND\_NEXT这个寄存器里写个NOP，紧接着读取JS\_HEAD\_NEXT\_LO和JS\_HEAD\_NEXT\_HI看是否有一个是0
         1. 如果有，那么就证明idx0还没有完成，所以可以直接REMOVE idx1，并且STOP idx0
         2. 如果没有，这就说明idx0完成了（此处我不知道是不是有两种情况可以判断idx0完成，一个是JS\_COMMAND\_NEXT是0，另一个就是这个LO和HI都是0），这时候就需要STOP idx1
   2. 如果idx1还没有被提交到GPU上，那么就REMOVE idx1并且STOP idx0

2. 如果idx1不要被killed，那么直接送个STOP给idx0就可以

 如果idx0不需要被killed，只有idx1需要被抢占，那么还是需要判断idx1的STATE

1. 如果idx1还没有被提交到GPU，那么就直接REMOVE
2. 否则，还是需要读取JS\_COMMAND\_NEXT这个寄存器的状态
   1. 如果是0，那么就证明idx0已经完成了，这时候idx1已经开始跑了，STOP idx1之
   2. 如果不是，那么就是说idx0还在执行，就给JS\_COMMAND\_NEXT这个寄存器里写个NOP，紧接着读取JS\_HEAD\_NEXT\_LO和JS\_HEAD\_NEXT\_HI看是否有一个是0
      1. 如果有，那么就证明idx0还没有完成，所以可以直接REMOVE idx1，并且
      2. 如果没有，这就说明idx0完成了，这时候就需要STOP idx1

**这里感觉逻辑很复杂，其实非常简单，就是判断谁在GPU，谁不在，不在就直接dequeue不含糊。 如果已经到了GPU，那么就看在不在跑，在跑就STOP，不然直接REMOVE。**


