2019/9/19 證實放棄fastai框架，原因為社區不夠活躍，框架過度定制而功能不夠全面，以及不容易和其他同學協作。

---

*2019/4/8*

# 概览

通过浏览，扫清了边边角角，确定要看的部分包括：

[主教程](https://course.fast.ai/index.html)

[进阶教程·前沿技术(18年版本，新版未发布)](http://course18.fast.ai/part2.html)

[数学基础(fast.ai的理念是让普通人可以做AI，所以他们的教程比较易懂，但不会深入)](https://github.com/fastai/numerical-linear-algebra/blob/master/README.md)

[机器学习(fast.ai的理念是让普通人可以做AI，所以他们的教程比较易懂，但不会深入)](http://course18.fast.ai/ml.html)

可以参考的资源有：

[github](https://github.com/fastai/fastai/blob/master/README.md)

[文档](https://docs.fast.ai/)

[论坛](https://forums.fast.ai/)

# [主教程](https://course.fast.ai/index.html)

## Getting started

今天申请提高aws实例数量限额还没得到回复，所以先在83上做了GPU部分的实验，可以看到，GPU在深度学习方面要比CPU快得多。

![alt text](https://github.com/RayXu14/Tools/blob/master/img/CPUvsGPU.png)

这里我还接触了jupyter notebook/lab里面[%timeit](https://ipython.readthedocs.io/en/stable/interactive/magics.html#magic-timeit)这个命令。似乎是一个计时的方法。以后我可以考虑写程序时候计时变为标准化操作，这样可以得知各部分跑的时间，以进行优化。

## Server setup
本来谷歌那个是最好的，有2000美元免费额度。可惜中国大陆境内没有业务，而且GCP我不熟。我怕节外生枝，觉得还是在AWS EC2烧钱稳妥一点。

---
*2019/4/9*

配好了位于东京的p2.xlarge实例，和教程不同的是，我为了方便，采用了我自己的bypass-all安全组策略，并且为了省钱，使用spot实例，不过在竞价方面把最高价设定为和按需使用的价格一样高。

至于jupyter notebook我用的是jupyter lab且在windows mobaXterm采用使用远程连接，具体可看
https://blog.csdn.net/cc1949/article/details/79095494

### 翻车

用spot实例**翻车**了，被terminated，而且拿不回来。

重开，这次不用spot，但是仍然bypass-all（方便jupyter lab远程使用），而且fastai的那个环境使用conda create另开，避免影响base。

又发现一个**问题**， 存储空间75G居然才刚好够放环境？！快点删没用的东西，尤其是在conda info --env里面找一找，不然悔之晚矣。也许本次翻车是这个原因。

## Lessons
写下我的收获。由于我事先有基础，已知则不论。

###  idea
- [ ] 可以利用google image的数据做一个北大识花app
* Jereny Howard认为强化学习对大多数人都显然没有鸟用，迁移学习才是正道。
* Jereny希望把fast.ai做成不需要coding的软件，让普通人都能够做人工智能。
- [ ] Jereny使用随机森林算法寻找最佳超参数，这是AutoML的内容。

### 底层工具
- [x] jupyter的工具确实不会清除之前占用的显存。notebook里面的垃圾回收：令模型=None，然后gc.collect()即可，实测在需要用的时候，pytorch就会延迟回收显存。这样可以避免重开notebook的麻烦。
- [x] fp16有时候结果还更好

### 数据处理
- [x] 数据处理类
    * DataBlock的分离机制我还挺喜欢的。DataBlock的更多作用：
      * 指定预处理
      * 通过制定类别/浮点列表决定是回归问题还是分类问题
    * DataBunch类基于pytorch的DataLoader。
- [x] 迁移学习是“永恒不变的热点”，没有理由不使用预训练数据。
- [x] 没有理由不利用全部数据。

### 调模型
- [x] 建议先用小数据尝试不同方法的优劣，再训全部数据集，以节省时间。
- [x] train loss和valid loss的关系
    - * train loss > valid loss 意味着欠拟合 ->可以增加训练轮次，减小后面的学习率，实在不行只能降低正则化。
    - * train loss < valid loss 不意味着过拟合，准确率下降才意味着过拟合。
- [ ] loss散度（波动范围）很大时，参数要么很小要么很大，无法继续训，要从头调学习率再来。
- [ ] 迁移学习要用discriminate learning rate才能取得如此优异的结果。
- [ ] 寻找最佳学习率
    * 解冻前：最大下降斜率处为选择的（最大）学习率，
    * 解冻后：为（最小）学习率，难找的就用最低点除以10（事实上解冻前也是这么选的），
    最大学习率为原来的1/5到1/10
- [x] Adam比SGD好，是momentum+RMSprop。动态学习率，然而还是需要设置学习率。还是得设置学习率退火。不过learner会代劳这些所有事情。
momentum就是动量加权（移动平均梯度），在SGD函数里面一般设置为0.9。
- [x] fit_one_cycle里面学习率先升高后降低时针对总batch数的，而不是每个epoch都升降一次。
    * 这个训练方法能使模型稳定训练并获得更好的泛化能力：一开始参数不对，梯度可能带偏；后期只能微小变化，不然过头。
    * fit_one_cycle的训练loss先大后小是好现象，意味着找到了合适的学习率。loss一开始就向下的话，需要担心陷入局部最优，影响泛化能力。可能需要调大学习率，挑出局部最优。
- [ ] 正则化。这些fastai库可以代劳，设置一下参数就好。
    - [x] weight decay（在我看来L2正则化的权重，但他说是做同一件事情的两个方面）
        * 避免过拟合，不一定要使用较少参数。可以用weight decay。因为参数很小的时候作用就可以很小的。
        * 未有0.1而不好的情况，0.1不需要early stop
        * 少数情况0.1难以拟合，需要0.01，0.01则需要early stop
    - [x] dropout。
      * pytorch dropout的时候会把比例除回来，确保量纲一致。
    - [ ] batch normalization
        * fast.ai实现的是移动平均的版本
- [ ] data augmentation

### CV
* 用小规模/小图片做预训练，可以得到更快的训练速度和更好的泛化能力。

### GAN
- [ ] train gan的时候，由于loss大致不会有大的偏差，所以很难判断结果好坏，基本只能按时查看一下生成的结果。

### 遗留问题
discriminate learning rate 最佳递增减幅度为2.6？对BERT是如此吗？

---
*2019/5/21*
# 缺陷和弥补

1. 框架过于细化，操作订得太死。使用的过程中陆续察觉到这一点，但都觉得有救，最终促使我濒临放弃Fastai库的便利的是，Learner类里面，一个batch的读取固定只能读取两个tensor。这实在令人无语。不过我又找到了BERT结合FastAI的方法，从 http://mlexplained.com/2019/05/13/a-tutorial-to-fine-tuning-bert-with-fast-ai/ 跳转到 https://github.com/deepklarity/fastai-bert-finetuning 。也许未来会真香吧。

--
*2019/5/29*

2. FastAI包装过头，本来我是想取而用之，不过从发现的这份代码来看，或者进行改装（类的继承）是更好的选择。
3. 输出过程中Lab断线问题就大了，图像都看不到了。但是事后在learner recorder的记录中可得。loss具体数值可以打印出learner.recorder.losses来看。吧。

--
*2019/6/26*

4. 逐渐发现FastAI有很强的定制性，其中以Callback的内容最为灵活。

--
*2019/7/24*

5. find_lr就算有suggestion也不是那么有用，姑且我选择的是最低点之前位置的学习率，以避免学不到最佳值，但是不追求最快收敛了

---
*2019/7/29*

6. 框架在loss_batch处钉死了输入是单个tensor，这连处理mask都很麻烦，而且找不到相关处理资料。**论坛里面回答对我来说又很慢**(这一点很致命，我提问题没人回答)。pytorch1.1已经出了Cyclical Learning Rate，于是我决心逐步放弃fastai

---
*2019/7/29*

7. 默认情况下learner并不会选择最优的checkpoint保存
