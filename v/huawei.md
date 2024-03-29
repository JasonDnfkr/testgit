### 主要工作
参与华为云教育板块：云学堂的后端研发工作，参与在线实验沙箱相关需求的开发。目前是在做沙箱系统的多租接口改造的一个工作；之前也是一直在沙箱这一块完成一些业务需求，比如用户参加考试时的权限与状态的判断；将整个项目里不同的时间字段格式统一为 UTC 格式；历史评论审核内部接口实现等一些需求等等。
主要是学习到了后端开发需求分析、代码编写、上线联调的全过程。

> I was involved in the backend development work for Huawei Cloud's education sector, specifically with Cloud School (云学堂). I participated in the development of online lab sandbox-related requirements. 
>
> My main contributions included implementing the historical comment review and cleaning interface, unifying backend timestamps to UTC format, refactoring legacy project code, and working on several business requirements such as multi-tenancy interface transformation. 
>
> I gained familiarity with the entire process of backend development, including requirement analysis, coding, and online debugging and testing.

该项目是华为云面向开发者市场的在线软件服务产品,旨在让开发者使用华为云的开发环境,学习相关的开发课程,并利用华为云提供的相关资源,开发其自身需求的软件业务,开拓华为云生态市场。

> This project is an online software service product by Huawei Cloud targeting the developer market. It aims to enable developers to use Huawei Cloud's development environment, learn relevant development courses, and leverage resources provided by Huawei Cloud to develop their own software solutions, thereby expanding Huawei Cloud's ecosystem market.

### 在做哪些需求？
#### 1. 沙箱多租接口改造

这个项目最初是为某些公司或机构定制的。现在，我们准备将其核心能力提取出来，作为一个可重复使用的解决方案。

> This project was originally customized for certain companies or organizations. Now, we are preparing to extract its core capabilities as a reusable solution.



目前在做的需求是：沙箱多租的接口改造任务，主要涉及到两点：

> The current requirement I am working on is the interface transformation task for sandbox multi-tenancy.

1. 对微服务的业务功能进行进一步划分：改造接口，将业务划分为编排层和原子层。目前存在的问题是，一，原子层的功能有一些混杂，粒度不够小（比如原子层内目前存在一些问题，比如有一些接口属于 update，应该切割为 select 和 update）；二，是把编排层的逻辑切割成原子层。
2. 给目前存在的接口，加上租户鉴权的一个业务逻辑。将记录的数据加上租户的字符串标识，存入新的数据库表里。也就是给当前用户查询出来的记录，加上tenant，userId的唯一标识。比如，有个场景是：每一个实验都有一个表记录相关数据，现在需要做的一个工作是，根据userId查询相应的实验记录，加上租户字符串标识，存入新的带租户字段的表中，做到一个数据转移更新的功能。

#### 2. 用户参加考试时的权限判断
用户可以直接访问接口URL来进入考试页面，此时需要有机制判断两个情况，一是该考试是否在进行中（用户是否开启了这场实验），二是该用户是否具备进行考试的权限。
一：如果用户在该场考试中，则允许用户继续考试；否则查询预约信息，判断是否能参加本次考试（用户是否预约，是否迟到，考试是否开始）
二：权限涉及到普通用户、实验作者、管理员等。如果是管理员等，则可以直接进入考试，否则需要查询预约信息。

#### 3. UTC 时间改造
当前沙箱项目中，储存时间的字段不统一，有的是 timestamp 时间戳，有的是字符串类 (varchar)，有的是 date 类型，并且时区不统一。
工作：
主要是对相关增删改查的接口进行修改，将时间修改为 数据库 datetime 类型。和前端约定，传参时，传入一个时区的偏移量，记录不同时区的偏移。
具体效果：
1. 后端打印日志时，要按照微服务的时区进行日志输出；
2. 将数据存入数据库时，要按照 UTC 时间，去掉偏移量。
3. 数据库连接时改为 datetime 类型。
4. 需要注意的点是，在人机面交互时，需要注意时区；机机面的交互，不需要转换。（还有一种情况是，比如原项目的机机面中有些参数的传递是时间戳类型，这个不需要处理，因为不涉及前端的数据展示和数据库存储）

#### 4. 历史评论审核接口
实现了一个内部接口，对已存在的历史评论（万级数量）进行审核，根据审核结果对评论进行处理，隐藏、删除或不处理。采用了一些小优化来处理这个需求，比如分批查询、计数器算法避免触发流控。

> I implemented an internal interface for reviewing a large number of existing historical comments, which numbered in the tens of thousands. Based on the review results, the comments were processed by either hiding, deleting, or leaving them unchanged. To handle this requirement more efficiently, I applied some optimizations such as querying in batches and using a counter algorithm to avoid triggering rate limiting.

如何做审核：
调用华为的大模型能力对评论正文进行审核，将评论内容以POST形式访问审核接口，会返回该条评论的风险等级（低，中，高）。

内部接口的作用是对已有的所有评论进行清洗审核，主要的思路是从数据库中查询出所有的评论，并依次调用审核接口，使用大模型的能力来获取每一条评论的风险等级。

由于审核机制可能会更新，因此每次审核清洗时，都要保存一个版本号作为标识。若在以后的需求中，如果审核机制发生变化导致判断结果不一致，则需要准确更新结果。

由于数量较大，因此要采取一定的优化手段：
1.每次调用该接口时，只会查询一定数量的评论（比如3000条），下一次调用该接口时，查询第3000~6000条，以此类推，避免一次查询过多数量的记录导致系统负载过大。

2.在调用大模型审核接口时，为了避免触发流量控制，需要采用一定的限流算法进行维护。由于是内部接口，不需要考虑用户感知方面的问题，采用了最朴素的固定窗口限流算法，限制1s内只能调用5次该接口。

3.为了防止接口被反复调用产生并发问题，使用了 redis 并发锁来防止接口被重复调用。

> 
> Each time the interface is called, only a certain number of comments are queried (e.g., 3000 comments). The next time the interface is called, it queries comments from 3000 to 6000, and so on, to avoid overloading the system with too many records in a single query.
>
> When calling the large model review interface, in order to avoid triggering rate limiting, a certain rate limiting algorithm is used for maintenance. Since this is an internal interface, there is no need to consider user perception issues, so we employed the most basic fixed window rate limiting algorithm, limiting the interface to be called only 5 times within 1 second.
>
> To prevent concurrent issues caused by the interface being repeatedly called, we used a Redis concurrent lock to prevent the interface from being invoked repeatedly.

https://blog.csdn.net/billgates_wanbin/article/details/123556273
