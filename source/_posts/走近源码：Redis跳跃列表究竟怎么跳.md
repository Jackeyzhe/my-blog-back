---
title: 走近源码：Redis跳跃列表究竟怎么跳
date: 2019-04-18 23:59:10
tags: Redis
comments: true
---

在前面介绍压缩列表ziplist的时候我们提到过，zset内部有两种存储结构，一种是ziplist，另一种是跳跃列表skiplist。为了彻底理解zset的内部结构，我们就再来介绍一下skiplist。<!-- more -->

#### skiplist介绍

顾名思义，skiplist本质上是一个有序的多维的list。我们先回顾一下一维列表是如何进行查找的。

![一维有序列表](https://res.cloudinary.com/dxydgihag/image/upload/v1555344398/Blog/Redis/skiplist/%E5%88%97%E8%A1%A8.png)

如上图，我们要查找一个元素，就需要从头节点开始遍历，直到找到对应的节点或者是第一个大于要查找的元素的节点（没找到）。时间复杂度为O(N)。

这个查找效率是比较低的，如果我们把列表的某些节点拔高一层，例如把每两个节点中有一个节点变成两层。那么第二层的节点只有第一层的一半，查找效率也就会提高。

![双层列表](https://res.cloudinary.com/dxydgihag/image/upload/v1555345563/Blog/Redis/skiplist/double_list.png)

查找的步骤是从头节点的顶层开始，查到第一个大于指定元素的节点时，退回上一节点，在下一层继续查找。

例如我们要在上面的列表中查询16。

- 从头节点的最顶层开始，先到节点7。
- 7的下一个节点是39，大于16，因此我们退回到7
- 从7开始，在下一层继续查找，就可以找到16。

这个例子中遍历的节点不比一维列表少，但是当节点更多，查找的数字更大时，这种做法的优势就体现出来了。还是上面的例子，如果我们要查找的是39，那么只需要访问两个节点（7、39）就可以找到了。这比一维列表要减少一半的数量。

为了避免插入操作的时间复杂度是O(N)，skiplist每层的数量不会严格按照2:1的比例，而是对每个要插入的元素随机一个层数。

随机层数的计算过程如下：

- 每个节点都有第一层
- 那么它有第二层的概率是p，有第三层的概率是p*p
- 不能超过最大层数

Redis中的实现是

```c
/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

其中ZSKIPLIST_P的值是0.25，存在上一层的概率是1/4，也就是说相对于我们上面的例子更加扁平化一些。ZSKIPLIST_MAXLEVEL的值是64，即最高允许64层。

#### Redis中的skiplist

Redis中的skiplist是作为zset的一种内部存储结构

```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

可以看到zset是由一个hash和一个skiplist组成。

skiplist的结构包括头尾指针，长度和当前跳跃列表的层数。

而在zskiplistNode，也就是跳跃列表的节点中包括

- ele，即节点存储的数据
- 节点的分数score
- 回溯指针是在第一层指向前一个节点的指针，也就是说Redis的skiplist第一层是一个双向列表
- 节点各层级的指针level[]，每层对应一个指针forward，以及这个指针跨越了多少个节点span。span用于计算元素的排名

了解了zset和skiplist的结构之后，我们就来看一下zset的基本操作的实现。

#### 插入过程

前面我们介绍压缩列表的插入过程的时候就有提到过skiplist的插入，在zsetAdd函数中，Redis对zset的编码方式进行了判断，分别处理skiplist和ziplist。ziplist的部分[前文](https://jackeyzhe.github.io/2019/03/23/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9A%E5%8E%8B%E7%BC%A9%E5%88%97%E8%A1%A8%E6%98%AF%E6%80%8E%E6%A0%B7%E7%82%BC%E6%88%90%E7%9A%84/)已经介绍过了，今天就来看一下skiplist的部分。

```c
if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
    zset *zs = zobj->ptr;
    zskiplistNode *znode;
    dictEntry *de;

    de = dictFind(zs->dict,ele);
    if (de != NULL) {
        /* NX? Return, same element already exists. */
        if (nx) {
            *flags |= ZADD_NOP;
            return 1;
        }
        curscore = *(double*)dictGetVal(de);

        /* Prepare the score for the increment if needed. */
        if (incr) {
            score += curscore;
            if (isnan(score)) {
                *flags |= ZADD_NAN;
                return 0;
            }
            if (newscore) *newscore = score;
        }

        /* Remove and re-insert when score changes. */
        if (score != curscore) {
            znode = zslUpdateScore(zs->zsl,curscore,ele,score);
            /* Note that we did not removed the original element from
             * the hash table representing the sorted set, so we just
             * update the score. */
            dictGetVal(de) = &znode->score; /* Update score ptr. */
            *flags |= ZADD_UPDATED;
        }
        return 1;
    } else if (!xx) {
        ele = sdsdup(ele);
        znode = zslInsert(zs->zsl,score,ele);
        serverAssert(dictAdd(zs->dict,ele,&znode->score) == DICT_OK);
        *flags |= ZADD_ADDED;
        if (newscore) *newscore = score;
        return 1;
    } else {
        *flags |= ZADD_NOP;
        return 1;
    }
}
```

首先是查找对应元素是否存在，如果存在并且没有参数NX，就记录下这个元素当前的分数。**这里可以看出zset中的hash字典是用来根据元素获取分数的。**

接着判断是不是要执行increment命令，如果是的话，就用当前分数加上指定分数，得到新的分数newscore。如果分数发生了变化，就调用zslUpdateScore函数，来更新skiplist中的节点，另外还要多一步操作来更新hash字典中的分数。

如果要插入的元素不存在，那么就直接调用zslInsert函数。

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    level = zslRandomLevel();
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```

函数一开始定义了两个数组，update数组用来存储搜索路径，rank数组用来存储节点跨度。

第一步操作是找出要插入节点的搜索路径，并且记录节点跨度数。

接着开始插入，先随机一个层数。如果随机出的层数大于当前的层数，就需要继续填充update和rank数组，并更新skiplist的最大层数。

然后调用zslCreateNode函数创建新的节点。

创建好节点后，就根据搜索路径数据提供的位置，从第一层开始，逐层插入节点（更新指针），并其他节点的span值。

最后还要更新回溯节点，以及将skiplist的长度加一。

这就是插入新元素的整个过程。

#### 更新过程

了解了插入过程以后我们再回过头来看更新过程

```c
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    /* We need to seek to element to update to start: this is useful anyway,
     * we'll have to update or remove it. */
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < curscore ||
                    (x->level[i].forward->score == curscore &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    /* Jump to our element: note that this function assumes that the
     * element with the matching score exists. */
    x = x->level[0].forward;
    serverAssert(x && curscore == x->score && sdscmp(x->ele,ele) == 0);

    /* If the node, after the score update, would be still exactly
     * at the same position, we can just update the score without
     * actually removing and re-inserting the element in the skiplist. */
    if ((x->backward == NULL || x->backward->score < newscore) &&
        (x->level[0].forward == NULL || x->level[0].forward->score > newscore))
    {
        x->score = newscore;
        return x;
    }

    /* No way to reuse the old node: we need to remove and insert a new
     * one at a different place. */
    zslDeleteNode(zsl, x, update);
    zskiplistNode *newnode = zslInsert(zsl,newscore,x->ele);
    /* We reused the old node x->ele SDS string, free the node now
     * since zslInsert created a new one. */
    x->ele = NULL;
    zslFreeNode(x);
    return newnode;
}
```

和插入过程一样，先保存了搜索路径。并且定位到要更新的节点，如果更新后节点位置不变，则直接返回。否则，就要先调用zslDeleteNode函数删除该节点，再插入新的节点。

#### 删除过程

Redis中skiplist的更新过程还是比较容易理解的，就是先删除再插入，那么我们接下来就看看它是如何删除节点的。

```c
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;
}
```

删除过程的代码也比较容易理解，首先按照搜索路径，从下到上，逐层更新前向指针。然后更新回溯指针。如果删除节点的层数是最大的层数，那么还需要更新skiplist的level字段。最后长度减一。

#### 总结

skiplist是节点有层级的list，节点的查找过程可以跨越多个节点，从而节省查找时间。

Redis的zset由hash字典和skiplist组成，hash字典负责数据到分数的对应，skiplist负责根据分数查找数据。

Redis中skiplist插入和删除操作都依赖于搜索路径，更新操作是先删除再插入。

#### 推荐阅读

Skip Lists: A Probabilistic Alternative to Balanced Trees

《[Redis 深度历险：核心原理与应用实践](https://juejin.im/book/5afc2e5f6fb9a07a9b362527)》