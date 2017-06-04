
# 赛题及数据描述描述

### 赛题描述

本题给出了9月19日到10月18日期间3个收费站共五个方向（其中2号收费站只有0方向）的车辆通过记录作为训练集。预测集中给出了10月19日到10月25日共七天
每天6点到8点，15点到17点的车辆通过记录，需要预测当天8点到10点，17点到19点以20分钟为单位的车流量

### 数据定义

description of the feature:

Traffic Volume through the Tollgates

| time          | datatime       | the time when a vehicle passes the tollgate                                                   |
| ------------- |:--------------:| --------------------------------------------------------------------------------------------- |
| tollgate_id   | string         | ID of the tollgate                                                                            |
| direction     | string         | 0:entry, 1:exit                                                                               |
| vehicle_model | int            | this number ranges from 0 to 7, which indicates the capacity of the vehicle(bigger the higher)|
| has_etc       | string         | does the vehicle use ETC (Electronic Toll Collection) device? 0: No, 1: Yes                   |
| vehicle_type  | string         | vehicle type: 0-passenger vehicle, 1-cargo vehicle                                            |


# 解题思路

- 数据中有部分车辆没有“vehicle_type”信息，通过分析数据发现：记录在案的载重量大于等于5的车辆都是货车，载重量为4的车辆以客车居多，载重量3，2的车辆货车居多，
载重量1的车辆客车居多。剩下载重量为0的车辆，不是很理解什么样的车辆载重量会是0，所以单独处理。所以“vehicle_type”为空的记录根据其载重量以及相同载重量的记录中较多的“vehicle_type”去填充空值。具体填充如下：

```python
vehicle_model0 = volume_df[volume_df['vehicle_model'] == 0].fillna("No")
vehicle_model1 = volume_df[volume_df['vehicle_model'] == 1].fillna("passenger")
vehicle_model2 = volume_df[volume_df['vehicle_model'] == 2].fillna("cargo")
vehicle_model3 = volume_df[volume_df['vehicle_model'] == 3].fillna("cargo")
vehicle_model4 = volume_df[volume_df['vehicle_model'] == 4].fillna("passenger")
vehicle_model5 = volume_df[volume_df['vehicle_model'] >= 5].fillna("cargo")
```

- 每个收费站和方向分开分析，分别有：1号收费站0方向；1号收费站1方向；2号收费站0方向；3号收费站0方向；3号收费站1方向。每块数据按照20分钟切片，
统计20分钟内通过的车流数量。以两个小时（6段）车流量为特征，用最后一个特征向后偏移20 * offset 时段的车流为因变量构建预测集，其中offset = 1，2，3...6。构建过程类似时间窗每次向后平移20分钟。例如：
2017年9月19日为例，时间切片之后车流数据如下：
| 时间段          | 车流量 |
| --------------- |:------:|
|00:00:00~00:20:00| a      |
|00:20:00~00:40:00| b      |
|00:40:00~01:00:00| c      |
|01:00:00~01:20:00| d      |
|01:20:00~01:40:00| e      |
|01:40:00~02:00:00| f      |
|02:00:00~02:20:00| g      |
|02:20:00~02:40:00| h      |
构建训练集:
| feature1 | feature2 | festure3 | feature4 | feature5 | feature6 | y |
|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|:-:|
| a        | b        | c        | d        | e        | f        | g |
| b        | c        | d        | e        | f        | g        | h |
这么做是因为预测集中需要用前两个小时预测后两个小时车流。尝试过不区分收费站和方向的情况，效果没有区分的好。

- 20分钟切片里除了有车流量，还有货车数量，客车数量，


# 部分提交思路以及结果

### 2017-05-11

- 在单模型的基础上加上一个清洗脏数据的功能，用三层gbdt清洗掉百分之十的数据，再用五层GBDT预测（不知道为什么才0.7876）

- 纯时间特征模型加上数值类型的时间特征（成功从0.2016提高到了0.1730）

- stacking1使用的训练集做微小调整，在构造训练集过程中不处理nan值，拿到模型里才处理，这样训练集的Volume0～volume5和y之间更有连贯性（0.1788已经被纯时间特征超过了）


### 2017-05-12

- 重新提交2017-05-11日第一个优化点（>0.3）

- 在2017-5-11的基础上减少stacking2模型的数量，纯时间模型去掉ADAboost(0.1775)

- 检查训练集，在2017-05-11的基础上减少stacking1模型的数据，去掉所有ADAboost模型，二层不适用随机模型，尽量减少模型之间的相关度（0.1735）

### 2017-05-13

- 在2017-05-12基础上修改过滤时的评分函数

- 修正时间特征计算代码0.1681

- stacking的第二层备选模型中去掉随机模型（ET和RF）(0.1739)

### 2017-05-15

- 在2017-05-13项目1的基础上，舍弃数据过滤功能，改成特征选取功能，功能按照单个特征做回归的评分高低降序排序，每次选取一个效果最好的特征加到备选集中，并预测打分，得到最好的特征集（0.1841降了）

### 2017-05-18 都是单模型调整

- GBDT3层和5层有没有区别，可以尝试一下，而且注意用评价函数（3层效果好，说明5层有严重的过拟合）

- 加一些特征吧，比如说最大值，最小值（0.1747）

### 2017-05-17 训练集和预测集数据分析

- 发现不论是原始的训练集还是原始的预测集，0方向（entry）的车辆都不记录vehicle_type，而1方向（exit）的方向都记录了该值，
所以不能盲目用众数来填充vehicle_type的空值，在分端口和方向的时候适当剔除掉一些特征

- 3层GBDT（0.1627）

- 存在7万条记录没有载重量，而且全部都是各收费站0方向，都没用电子桩（暂时不做处理，因为题目也说载重量是0到7，说明0载重量是存在的）

### 2017-05-18

- 在昨天基础上，删除处理后产生的空值，之前我都是在stacking文件里专门处理训练集和预测集的空值，在volume_predict2文件里
没有这功能。（0.1683）

- 在昨天基础上，增加层数到5层（0.1694）

过拟合了，想想怎么肥四

### 2017-05-19

- 把10月1日到10月7日的数据加回来，增加部分特征，包括20分钟内使用etc的车辆数、总载重量，没使用etc的车辆数、总载重量

### 2017-05-21

#### 两种筛选训练集方式

- 按照预测集分布筛选，用预测集车流的最大值，最小值，均值，标准差压缩训练集vloume0 ~ volume5的区间，注意用均值加减二倍标准差和
   最大值，最小值比较（0.1687）

- 直接用训练集本身的分布特点筛选数据，用训练集的最大值，最小值，均值，标准差压缩区间，均值加二倍标准差为上限，
   下限设置常数3（0.1692）

#### 之前单模型提交的数据结果及方式概要：

- 0.1672 分端口方向构造不同的特征，10月1日到10月7日数据全部变成null，但是当时忘记删除这些值，将其全部填成0

- 0.1747 在上面的基础上删除所有的空值

- 0.1647 在上面的基础上删除部分特征，包括最大值，最小值，均值等特征

- 0.1658 考虑到大量的0数据能够改变树的分布，将部分较小的数值隔离开，因此想增加10月1日至10月7日数据来平衡数据分布

- 0.1687 用测试集的最大值，最小值，均值，方差清洗数据，均值加减二倍方差

- 0.1692 用训练集的最大值，最小值，均值，方差清洗数据，均值加减二倍方差和最大值，最小值，用最大区间


### 2017-05-22

#### 暂时放弃上述两种数据清洗方式，使用如下数据清理方式：

- 10月1日至10月7日数据全部删除

- 除此之外，其余数据resample得到的空值均填为0

#### 待提交方法

- 用之前的非调参stacking模型+时间特征模型的融合（0.1728， 0.1617）

- 使用另一种stacking方式，二层固定使用两种参数的XGB和一个GBDT，一层使用三种模型集合，集合长度分别为6,7,6（0.1673，0.1602）

- 单模型调整，单独剔除10月1日到10月7日的数据（保证resample之后没有这个东西），在此基础上将所有的空值设置为0（0.1674）

### 2017-05-23

- 单模型调整，删除全部空值（尽量合理删除空值），顺便检查代码

- 把1放到05-22的第二个stacking模型中

- 时间特征模型调参，可以稍微过拟合，目的是降低和主train&test1模型的相关度，0.5x+0.5y的比例分配结果，最后成绩是0.1589



