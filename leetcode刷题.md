#### 数组

1. 双指针
2. 滑动窗口
3. int mid = left + ((right - left) / 2)
4. List.toArray(new int[len])
5. list.toArray(new int[list.size()[]])

#### 链表

1. 虚拟头节点 dummy   加减结点 记得size+/- !!!
2. 双指针
3. 虚拟头节点可以预防只有一个结点元素的测试用例
4. 

#### 哈希表

1. 看情况选择 list,   set,   map
2. 多指针方法在大脑具象化方法
3. LinkedList提供了Stack  Queue(offer  poll  peek)  Duque的接口; 直接用Deque的add, remove方法
4. StringBuild.deleteCharAt
5. Hashmap.getOrDefault   HashMap.computeIfAbsent

#### 字符串

1. 掌握Java常用api   [java刷题必备api【常用】_做题api-CSDN博客](https://blog.csdn.net/King_wq_2020/article/details/118889283)
2. String.split   正则表达式：Java中  两个\\\才表示转义字符
3. \d 数字  \D非数字    \s任何空白字符     \w数字字母      大写表示非^
4. indexOf()  对应KMP：KMP的主串指针是一直往前的，调整的是匹配串的指针
5. 字符串翻转借助StringBuilder.reverse()
6. Character.isLetterOrDigit   Character.toUpperCase
7. str.endsWith
8. 一定要用equals不能用==
9. StringBuilder   append   charAt   deleteCharAt
10. PriorityQueue是queue，所以方法也是offer,  poll,  peek，只不过顺序是自己指定的
11. PriorityQueue<>的<>不能省略

### 树

1. 返回值boolean的递归，&&
2. 前序后序 高度深度？   前序求深度，后序求高度
3. **后序遍历是利用到递归的结果相加/相乘，前序遍历是不利用后序递归的结果，所以只能在条件上做改变；所以前序遍历往往需要添加一个新的参数用来辅助判断或者构造中间结果
4. 后序遍历，分而治之的思想，或者直接层次遍历判断条件求值
5. 如果终止条件不需要处理过程，直接判断非终止过程即可
6. 终止条件是指递归终止，是结点为null，判断结点是否相等的代码逻辑不属于终止条件判断范畴
7. stream()  filter,  distinct,  limit,  skip,  map,  mapToInt,  sorted     count, forEach,  reduce,  collect(Collectors.toList())   toArray
8. mapToInt(Integer::intValue  /  String::length)
9. 分情况多次判断时，在超出终止条件(代码写多余的情况下)，有些多余的情况(在终止条件之前的情况)要直接return就可以，否则会因程序多余情况判断导致出错
10. 从中序和后序遍历序列构造二叉树，构造的头节点一直是后序遍历数组的最后一个，而不是每次的right

### 回溯

1. backtracing(n,k, 回溯的起始值i或者是count)，回溯的是深度，在回溯的同一层遍历的是所有可能的结果
2. 递归用来纵向遍历，for循环用来横向遍历
3. 递归思考的是下一步的解决方案，循环思考的是现阶段的解决方案；脑海里要有树的方案图
4. 当前拿走1，下一步拿走2，3；或者当前拿走2，下一步拿走3
5. 对‘abcd’, 当前拿走a, 下一步拿走b/bc/bcd，或者当前拿走ab， 下一步拿走c/cd
6. 思考过程，这一步能干啥，下一步要干啥
7. 子集问题是收集所有树结点，而组合问题，分割问题是收集树的叶子结点
8. 不能包含重复的子集代表同一层遍历的时候不能出现连续的数字，递归可以出现连续的数字(排序后)，因为是组合没有顺序
9. s.charAt(i)-‘0’  字符转数字，可用于数组的下标计算
10. 除了排序比较nums[i]==nums[i-1]方法外，还可以定义一个boolean flag=[false]，标记有没有访问过
11. 不可以直接用set判断当前元素是否已经在本次迭代中使用，会出现414    441   无法保证在深度回溯的时候发生顺序不同的组合；所以只能在要求顺序的时候使用set，如非递减子序列

#### 贪心算法

1. 先判断i<j 再 g[i]<s[j]
2. Arrays.Stream(nums).sum()
3. 

#### 动态规划

1. 背包问题可以用一维数组或者二维数组，一维数组必须从后往前遍历，否则会同一个物品被装入多次

2. 01背包本质就是分东西，分成两堆石头，达到离我期望的目标(value，石头质量)最近的区分结果，期望是均匀分层两堆质量相近的石头；  别管怎么分，反正一定满足在期望的value上限内，配置方案实现最接近value的值(01背包，每个元素只参与一次)

3. 两堆是代表选中和没选中

4. Arrays.stream().sum()比直接for循环求sum慢得多

5. 背包问题：容量为i的背包最多能装多少价值的东西;    

6. 组合问题: 装满有几种方法  dp[j] += dp[j - nums[i]]，记住此时要初始化dp[0]=1；

7. 求硬币个数最少的问题才是dp[j-nums[i]]+1，一个求组合数，一个求到达target所使用的最小物品个数，而非组合数;把value当作个数1

8. **如果求组合数就是外层for循环遍历物品，内层for遍历背包**。

   **如果求排列数就是外层for遍历背包，内层for循环遍历物品**。

9. 0/1背包只能先物品再背包容量，完全背包都可以，但是物品要从小到大保证物品可以拿多次；组合(完全背包)则必须先背包容量再物品，保证物品是有顺序的

10. 完全背包/01背包是背包容量大小顺序的区别，组合和排列是先背包后物品的顺序的区别

11. 打劫(二叉树)或者买卖股票分成两种状态  打劫/不打劫；持有/不持有，dp的数组都可以设置为2，分别更新两种状态

12. 买卖股票用数组为2的dp表示持有/不持有，打家劫舍也可以用数组为2，区别在于买卖股票的dp 0和1是连续的事件，可以一天之内选择买卖，打劫不是计算dp[1]用到的dp[0]应该是上一阶段的dp[0]，不能使用本轮更新过的dp[0]

13. 一定先要明确dp数组代表的含义，对于最长递增子序列，dp[i]代表的是以i结尾的数组子序列最大长度，而不是整个数组子序列的最大长度，所以还需要比较每一个i结尾的长度

14. 分清楚求公共最长子序列还是公共最长连续序列，子序列if(num1[i]==num2[j])dp\[i][j]=dp\[i-1][j-1]+1；else  dp\[i][j] = max(dp\[i-1][j],  dp\[i][j-1])； 连续子序列只有if； 并且还得分清楚dp代表的含义，子序列是可以传递到dp\[len][len]的，连续子序列一般指的是以len结尾的最长子序列长度而非所有的最长子序列，所以需要逐一判断最长

15. s中包含t的子序列个数  和  s和t中子序列的最大长度  注意区别,dp\[i][j]的更新顺序箭头方向不一样，仔细辨别其中区别

16. 编辑距离：子序列问题终极绝杀，包括删除操作和替换操作；删除是dp\[i][j]=dp\[i-1][j]+n；替换是dp\[i][j] = dp\[i-1][j-1]+n

17. 

#### 单调栈

1. 求下一个更大的元素，即用单调栈解
2. 

#### 图论

1. bfs将未访问的节点加入队列时就应该标记其已访问，否则会重复加入同一节点
2. 并查集：find,  join,  isSame,  int[] parent
3. parent针对的是所有节点，先初始化parent，然后对每一条边(即两个节点)遍历，满足条件就join，最后得到不同交集
4. 回溯需要回退的一般还是path的记录，像改动0为1的话就不存在回退的问题
5. dfs是遍历每一个节点，访问过的节点不需要再访问，回溯是寻找每一条可能的路径；dfs的四个方向已经保证在该点的全部dfs路径，所以不用担心没有回溯，下一次遇到该点visited为true就返回造成路径搜索没有覆盖全部的问题：错误理解
6. 回溯算法属于dfs，区别是是否利用回溯记录每一条路径path，dfs仅仅是遍历每一个节点不需要通过回溯记录；举例：[1 2 3 4]的组合，dfs直接一条路径1234就完结了，回溯是要求所有组合，所以需要visited重新设置为false的操作
7. 



####  实习二刷

1. 自己编写链表时，头节点+size，并且记得size++
2. 判断链表相交节点时，除了双指针，用HashSet
3. set转数组，利用 set.stream().mapToInt(i -> i).toArray()
4. 用两个Stack模拟Queue，在InStack里push，OutStack里pop，outStack为空就把inStack搞过来
5. 用两个Queue模拟Stack，queue1是主队列，queue2用来备份，当push时，先offer给2，再把1全给offer到2，交换1/2，始终用queue1来执行其他操作
6. PriorityQueue<int[]> pq = new PriorityQueue<>((pair1,pair2)->pair2[1]-pair1[1]);
7. for(Map.Entry<Integer,Integer> entry:map.entrySet())
8. LinkedList没有实现stack的接口，直接调用deque就行
9. 二叉树采用回溯的方法，也是以前序后序方式处理
10. 二叉树回溯时，终止条件那只处理root==null return，其他用前序逻辑处理
11. 二叉搜索树找最近的公共祖先，判断当前节点是否介于两节点值之间，先判断容易的if条件，再直接else判断复制if条件，if(<&&<)  elseif(>&&>) else break;
12. 回溯中组合的剪枝:for(int i=0; i<n-(k-path.size())+1; i++)
13. 递归思考的是下一步的解决方案，循环思考的是现阶段的解决方案；脑海里要有树的方案图
14. digist.charAt(i)-‘0’     StringBuilder.length()
15. 没有Arrays.sum方法，用stream流
16. List, Set转int[] 都需要用Stream流 mapToInt
17. 背包问题可以用一维数组或者二维数组，一维数组必须从后往前遍历，否则会同一个物品被装入多次
18. 01背包本质就是分东西，分成两堆石头，达到离我期望的目标(value，石头质量)最近的区分结果，期望是均匀分层两堆质量相近的石头
19. 



#### 笔试提示词

1. 你是一个精通数据结构与算法、具有极强算法思维和代码编写能力的JAVA程序员，我将在之后给出多道代码笔试题，代码笔试题将分为问题描述、输入描述、输出描述、测试样例，你需要使用Java语言编写符合代码笔试题要求的代码，并以ACM模式在代码框中输出代码。请通过如下步骤完成对代码的编写，步骤：1.分析题目，完成代码编写；2.检查编写的代码是否能通过测试样例的检测，如果不通过，继续回到步骤1重新编写代码，直至能通过测试样例。输出代码要求：尽量降低代码的时间复杂度，以ACM模式在代码框中输出。下面我将给出一个示范

```java
class Singleton{
    private static volatile Singleton singleton;
    private Singleton(){};
    public static Singleton getInstance(){
        if(singleton==null){
            synchronized(Singleton.class){
                singleton = new Singleton();
                return singleton;
            }
        }else{
            return singleton;
        }
    }
}
```

