---
title: 王小锤学Java：retainAll函数那点儿事
date: 2019-03-13 22:55:41
tags: 技术杂谈
comments: true
---

王小锤是一家电商网站的Java程序员，下午刚打开电脑，公司的运营妹子小美就过来找他：“小锤，你能帮我导一份数据吗？我需要昨天成为SVIP用户，并且之前给过差评的这些账号，不过要把一个叫大宝的账号去掉，他老板亲戚。”对于小美的需求，小锤从来没有拒绝过，这次也不例外。<!-- more -->

答应小美之后，小锤心想，这么简单的事情，一个SQL就搞定了。小美又要夸我效率高了，想想还有点小激动呢。小锤打开电脑的一瞬间整个人都不好了。原来今天早上下班前，他才刚刚完成微服务的拆分。现在订单和用户已经分成了两个服务，评价表和用户等级表也已经在两个数据库里了。这就尴尬了，本来一个SQL能解决的事情，现在还需要跨服务调接口了。

不过小锤马上就有思路了：

![小锤的思路](https://res.cloudinary.com/dxydgihag/image/upload/v1552493631/Blog/Other/%E5%B0%8F%E9%94%A4%E5%B7%A5%E4%BD%9C%E6%B5%81.png)

有了思路之后，小锤立刻就开始写代码了。

两分钟后……新鲜的代码出炉了（这里给个类似的Demo）。

``` java
public class NoBadReviewSVIP {
    public static void main(String[] args) {
        String[] svipArr = new String[]{"大宝","凉凉","Tom"};
        String[] badReviewUserArr = new String[]{"凉凉","Tom","大宝","小锤"};
        List<String> svips = new ArrayList<String>(Arrays.asList(svipArr));//从用户服务查
        svips.remove("大宝");
        List<String> badReviewUsers = new ArrayList<String>(Arrays.asList(badReviewUserArr));//从订单服务查
        if (svips.retainAll(badReviewUsers)) {
            System.out.println(svips);
            System.out.println("有交集");
        } else {
            System.out.println("没有交集，告诉小美");
        }
    }
}
```

小锤先运行了一下，发现结果是没有交集，于是小锤告诉小美：“没有你要的用户，昨天新的SVIP都没有给过差评。”小美说：“不可能，老板说有一个叫Tom的用户就给了差评，你再好好查查。”

小锤一查发现Tom真的给了差评，而且昨天刚刚成为SVIP。到底是哪里出了问题呢？小锤对着代码看了一个多小时也没发现问题，于是向身边的大锤请教。大锤只看了一眼，告诉他if的条件不对，让他看retainAll的具体实现。

#### retainAll的实现

于是小锤就开始看retainAll的源码了，发现它调用了一个batchRemove的私有方法。

``` java
    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        //w表示两个list的公共元素个数
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                //如果有公共元素，存入elementData
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // c.contains()抛异常，就把剩余元素复制到elementData
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                //此时w为调用方list的长度
                w += size - r;
            }
            //如果元素数量改变
            if (w != size) {
                // 多余空间教给GC
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                //记录list修改的次数
                modCount += size - w;
                //重新设置list大小
                size = w;
                //返回true
                modified = true;
            }
        }
        return modified;
    }
```

这么一看小锤就明白了，对于

``` java
listA.retainAll(listB)
```

listA中保存了listA和listB的交集，如果listA中元素数量改变，那么就返回true；如果listA的元素不变，则返回false。

也就是说，判断两个list是否有交集，只需要执行上述代码，然后判断listA的size是否大于0就可以了。

想通了这一点，小锤马上修改了代码，重新运行，得到正确结果，并且把结果告诉了小美。

万万没想到，小锤最后仍然没有得到小美的夸奖，因为所有的SVIP都给了差评，小美要负责挨个询问原因并且帮助他们解决问题，所以根本没有时间搭理小锤。

#### 日常自黑

![斐波那契程序员](https://res.cloudinary.com/dxydgihag/image/upload/v1552574758/Blog/Other/%E6%96%90%E6%B3%A2%E9%82%A3%E5%A5%91.jpg)

