<center><h2>
    中山大学数据科学与计算机学院本科生实验报告
    </h2><h4>
    (2019年秋季学期)
    </h4></center>

| 课程名称 | 区块链原理与技术 | 任课教师     | 郑子彬           |
| -------- | ---------------- | ------------ | ---------------- |
| 年级     | 大三             | 专业（方向） | 软件工程         |
| 学号     | 17308154         | 姓名         | 王俊焕           |
| 电话     | 13242890979      | Email        | 415391519@qq.com |
| 开始日期 | 2019-12-9        | 完成日期     | 2019-12-13       |

### 一、项目背景

​		现实生活中，企业之间会存在由于资金暂时短缺，从而在商品交易后未能及时形成资本的转移，这时候企业之间会签订应收账款的单据，作为之后还款的依据。同时由于这个单据可以受到别家企业或者金融机构的信任，所以这个单据也可以作为被欠款企业用于购买别家企业商品的资本，或者用于向金融机构借款。但是这份信任的赋予是与许多事务相牵连的，对于大型企业，金融机构对于企业的信用分析的评估是容易的，因为大型企业的影响力与其还款能力是容易被检验的。当单据的应收账款随着企业之间的二次购买（利用应收账款作为向别家企业购买的凭证），对于这些企业（常见为中小型企业），金融机构对他们的信用分析并不容易，需要金融机构投入一定的经济成本做验证分析的，这其中的问题就是因为这个应收账款的欠债方的信用无法在整个供应链中传递，以及这些二次交易（甚至多次）的交易信息不透明化所导致的。

​		将供应链上的每一笔交易和应收账款单据上链，同时引入第三方可信机构来确认这些信息的交易，例如银行，物流公司等，确保交易和单据的真实性。同时，支持应收账款的转让，融资，清算等，让核心企业的信用可以传递到供应链的下游企业，减小中小企业的融资难度。

![1576249063943](/img/1.png)

### 二、方案设计

#### 1. 存储设计

- 结构体Company：表示公司信息

  ```c++
  struct Company{
      string name;//公司名字
      address addr;//公司地址
      bool isExisted;//公司是否存在
  }
  ```

- 结构体Receipt：表示借据信息

  ```c++
   struct Receipt{
  	address borrower;//借款者地址
  	address lender;//贷款者地址
  	uint money;//借款金额
  	string info;//借款信息（原因）
  	uint date;//时间戳
  	bool isFinaced;//是否作为银行融资凭证
  	bool confirmed;//是否被Confimed
    }
  ```

- 映射addr2Company：表示地址到公司的对应

  ```javascript
  mapping(address=>Company)public addr2Company;
  ```

- 映射comp2Receipt：表示公司到与其相关的借据数组的对应。此处映射的借据数组，公司可能是借款方，也可能是欠款方。当建立欠款单据时，双方都会在映射中加入该记录。

  ```javascript
  mapping(address=>Receipt[])public comp2Receipt;
  ```

- 部署合约的用户地址（此处为银行部署合约）

  ```java
  address public bankAddr;
  construct() public{
      bankAddr = msg.sender;
      addr2Company[bankAddr] = Company("bank",bankAddr,0,true);
  }
  ```

#### 2. 数据流图

![2](2.png)

#### 3. 核心功能介绍

- 列出与当前公司的所有相关账单：根据给定的公司的地址公钥可以查阅与其相关的所有账单，该公司可能为借款方也可能为贷款方。

  ```java
  /*
  	获取当前公司相关的账单数目 
  	输入参数：address（公司的地址）
  	返回：uint（账单数目）
  */
  function getReceiptLength(address comp) public returns(uint){
       return comp2Receipt[comp].length;
   }
  
  /*
  	获取当前公司的第i条账单记录
  	输入参数：address（公司的地址）、uint（账单索引）
      输出参数：账单记录中的条目
  */
  function getReceipt(address comp, uint index) public returns( address borrower,
  				address lender,uint money, string info, uint date, 
              bool isFinaced, bool confirmed){
      Receipt rec = comp2Receipt[comp][index];
      return (rec.borrower,
              rec.lender,
              rec.money,
              rec.info,
              rec.date,
              rec.isFinaced,rec.confirmed);
  }
  ```

- 签发应收账款：在商品交易中，借款方向贷款方购入一定价值的商品但未能即时予以资本购入，所以列下应收账款作信用凭证。该活动过程将被上链记录。例如车企从轮胎公司购买一批轮胎并签订应收账款单据。 

  ```java
  /*
  	签发应收账款记录上链
  	输入参数：address（公司的地址）、uint（应收账款金额）、info（应收账款信息）
  */
  function buyAndCheck(address lender, uint money, string memory info) public returns(string){
      //判断是否有这几家公司，若没有则首先创建对应的键值对存入addr2Company中。
      address borrower = msg.sender;
      if(!addr2Company[borrower].isExisted)
      	addr2Company[borrower] = Company("", borrower, true);
      if(!addr2Company[lender].isExisted)
      	addr2Company[lender] = Company("",lender, true);
  
      //借款方和贷款方都要记录该条应收账款
      uint date1 = now; //记录时间戳
      comp2Receipt[borrower].push(Receipt(borrower, lender, money, info, date1,false,false));
      comp2Receipt[lender].push(Receipt(borrower, lender, money, info, date1,false,false));
  }
  ```

- 应收账款转让：多家公司在商品交易过程中，使用与别家公司的应收账款作为资本，即应收账款发生转移，需要计算借款方和贷款方之间的金额转移。轮胎公司从轮毂公司购买一笔轮毂，便将于车企的应收账款单据部分转让给轮毂公司。轮毂公司可以利用这个新的单据去融资或者要求车企到期时归还钱款。 

  ```java
  /*
  	应收账款转让，由需要转移应收账款的中间公司调用。即A、B（中间公司）、C间，原本为A欠B、B欠C，现在需要用A欠B的钱来抵用至B欠C中。
  	输入参数：address（A公司的地址）、address（C公司的地址）、uint（转移金额）、info（转移相关信息）
  */
  function transferCheck(address borrower, address newLender, uint money, string memory info) public{
      //borrower和oldLender之间的欠条金额减去money：
      //1.找到它们之间对应的欠条
      //2.金额计算
      uint index =  0;
      for(index = 0; index < comp2Receipt[borrower].length; index++){
          if(comp2Receipt[borrower][index].borrower == borrower && comp2Receipt[borrower][index].lender == msg.sender && comp2Receipt[borrower][index].money >= money){
              comp2Receipt[borrower][index].money -= money;
              break;
          }
      }
      for(index = 0; index < comp2Receipt[msg.sender].length; index++){
          if(comp2Receipt[msg.sender][index].borrower == borrower && comp2Receipt[msg.sender][index].lender == msg.sender && comp2Receipt[msg.sender][index].money >= money){
              comp2Receipt[msg.sender][index].money -= money;
              break;
          }
      }
      
      //oldLender和newLender之间的欠条减去money：找到对应的欠条
      for(index = 0; index < comp2Receipt[msg.sender].length; index++){
          if(comp2Receipt[msg.sender][index].borrower == msg.sender && comp2Receipt[msg.sender][index].lender == newLender && comp2Receipt[msg.sender][index].money >= money){
              comp2Receipt[msg.sender][index].money -= money;
              break;
          }
      }
      for(index = 0; index < comp2Receipt[newLender].length; index++){
          if(comp2Receipt[newLender][index].borrower == msg.sender && comp2Receipt[newLender][index].lender == newLender && comp2Receipt[newLender][index].money >= money){
              comp2Receipt[msg.sender][index].money -= money;
              break;
          }
      }
      
      //borrower和newLender之间的欠条金额加上money：直接新增一个欠条 info
      uint date = now;
      comp2Receipt[newLender].push(Receipt(borrower, newLender, money, info, date, false, false));
      comp2Receipt[borrower].push(Receipt(borrower, newLender, money, info, date, false, false));
  }
  
  ```

- 使用应收账款向银行融资，供应链上所有可以利用应收账款单据向银行申请融资。 

  ```java
  /*
  	输入参数：uint（融资金额）、uint（应收账款的索引）
  	若融资金额大于对应应收账款的金额数，则融资失败。
  */
  function borrowMoney(uint money, uint receipt_idx)public{
      if(comp2Receipt[msg.sender][receipt_idx].isFinaced)
          revert("The receipt have been used to finace.");
      if(comp2Receipt[msg.sender][receipt_idx].money < money)
          revert("The amount of money to be borrowed is greater than the limit");
      comp2Receipt[msg.sender][receipt_idx].isFinaced = true;
  }
  
  ```

- 应收账款的结算，应收账款单据到期时核心企业向下游企业支付相应的欠款。 

  ```java
  /*
  	输入参数：uint（应收账款的索引）
  	对于需要结算的账单，需要从借款方和贷款方的账单中进行删除（清零）
  */
  //function4: settle check
  
  function settleCheck(uint receipt_idx)public{
      Receipt storage rec = comp2Receipt[msg.sender][receipt_idx];
      uint index;
      bool flag = false;//判断是否settle
      for(index = 0; index < comp2Receipt[msg.sender].length; index++){
          if(comp2Receipt[msg.sender][index].borrower == rec.borrower && comp2Receipt[msg.sender][index].lender == rec.lender){
              if(comp2Receipt[msg.sender][index].money == rec.money){
                  uint index2;
                  address to = rec.lender;
                  uint date4 = now;
                  for(index2 = 0; index2 < comp2Receipt[to].length; index2++){
                      if(comp2Receipt[to][index2].borrower == rec.borrower && comp2Receipt[to][index2].lender == rec.lender){
                          if(comp2Receipt[to][index2].money == rec.money){
                              delete comp2Receipt[to][index2];
                              comp2Receipt[to][index2].date = date4;
                              comp2Receipt[to][index2].info = "settled";
                              break;
                          }
                      }
                  }
                  delete comp2Receipt[msg.sender][index];
                  comp2Receipt[msg.sender][index].date = date4;
                  comp2Receipt[msg.sender][index].info = "settled";
                  flag = true;
                  break;
              }
          }
      }
      if(!flag)
          revert("fake receipt!");
  }
  
  ```

### 三、功能测试

每个公司在使用该平台时都需要进行配置，修改当前自己该公司的公钥文件（在/Server/client_config.py，此处为随意更改，赋予权限修改的功能在此次没有上线），表示当前合约调用者为当前公司。在该平台中，每个公司都有自己的客户端与服务端，而用户的分配与管理是由公钥管理和分发机构负责，该平台不负责。所以我们默认，使用该平台时已经确定当前公司是哪一家公司。（所以没有登录选项）。对于查询账单，查询的是与当前公司相关的账单。其他功能都是以当前公司为发起者（所以功能中没有当前合约发送者私钥的待填项目）。

- 列出所有相关的账单：一开始显示暂无数据，点击更新加载后可能需要等待。（此时按钮会显示加载中，加载完毕后按钮再次显示回更新加载）。

  - 初始状态

  ![1576242507018](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576242507018.png)

  - 点击更新加载后显示【加载中】

  ![1576242522121](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576242522121.png)

  - 加载完毕

  ![1576240886114](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576240886114.png)

- 查询公钥地址：可以点击右上角看到当前已在链上有部署的所有公司的公钥地址。便于之后交易需要用到公钥地址时进行复制粘贴。

  ![1576243174063](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576243174063.png)

- 签发应收账款：从若公钥的输入为不存在的地址则会导致上链失败。其他符合输入条件并成功生成交易的输入会得到上链成功的消息提示，并且在查询账单中再次点击【更新加载】，可以看到上链成功记录。（此处为公司A向公司B借款4000）

  - 签发应收账款：上链失败

  ![1576243645292](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576243645292.png)

  - 签发应收账款：上链成功  

  ![1576243726522](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576243726522.png)

  - 查询账单：新增账单

  ![1576244076061](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576244076061.png)

- 应收账单转让：当前公司为中间公司，输入原借款公司公钥与当前贷款公司公钥进行转移。此处功能展示分为一下几步：

  - 搭建B公司的客户端与服务端，B公司向C公司借款2000。可以看到刚才A公司向B公司的借款记录也记录在B公司的相关账单中。

    ![1576245076572](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576245076572.png)

  - B公司作为中间公司，将A公司对自己的借款账单进行转移，对于以上数据的结果最终为：A公司欠款B公司2000，A公司欠款C公司2000，B公司不欠款C公司。

    |               B公司转移A公司对自己的的应付账单               |
    | :----------------------------------------------------------: |
    | ![1576245368682](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576245368682.png) |
    |   **转移结果**（B公司视角：与A公司、C公司的账单金额减少）    |
    |    ![](F:\软工大三上\区块链\区块链原理与技术大作业\3.png)    |
    | **转移结果**（A公司视角：与B公司账单金额减少，新增与C公司的借款） |
    |    ![](F:\软工大三上\区块链\区块链原理与技术大作业\2.png)    |

- 向银行融资：从查询的账单中得到用于融资的账单index（第一条index为0依次递增），融资后该账单的【是否用于融资】条目会显示YES。对于已经用于融资的账单融资将不成功。

  - 此处融资第2条，融资金额为3000

    ![1576246964988](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576246964988.png)

  - 结果查询显示为已用于融资。

    ![](F:\软工大三上\区块链\区块链原理与技术大作业\4.png)

- 支付清算：从查询的账单中得到用于支付清算的账单index（第一条index为0依次递增），清算后该账单的所有条目将清零，账单信息更新为“settled”，时间更新为清算的时间。对于已经清算的账单将不成功。

  - 输入清算的账单索引（此处为1）

    ![1576247333050](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576247333050.png)

  - 回到列出账单，结果被清零，信息修改为“settled”，并且时间戳也更新。

    ![1576247386276](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576247386276.png)

### 四、界面展示

- 主页面：可以看到最上部就展示了该项目的功能，右上角可查看所有公司的公钥地址
  ![1576240847242](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576240847242.png)

- 查看公钥地址

  ![1576243181363](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576243181363.png)

- 查询账单页面
  ![1576240886114](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576240886114.png)

- 签发应收账款页面

  ![1576240942909](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576240942909.png)

- 应收账款转让页面

  ![1576240986540](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576240986540.png)

- 银行融资页面

  ![1576241012844](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576241012844.png)

- 支付结算页面

  ![1576241039812](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1576241039812.png)

### 五、心得体会

​		本次项目是要我们搭建一个供应链系统金融平台，结合区块链实现应收账款的相关操作，借此感受一下区块链是如何应用于实际中，感受区块链的去中心化、共识机制。但是本次项目要我们去自己从实践中学习的东西真的太多了。从搭建fisco-bcos的环境、使用solidity编写智能合约、在webase上部署智能合约并实现交易，到基于fisco-bcos提供的sdk自己搭建一个这样的平台，对于一个一开始对区块链没什么了解，对Web开发的前端、后端也只是略懂皮毛的我来说真的很难很难。万事开头难，在部署环境、初次使用sdk时遇到的困难是最多的，总是会出现一些奇奇怪怪的bug。不过不得不说，基于已有框架去写代码，和以前学计算机基础课时需要自己造轮子还是不同的，基于框架轻松了很多。但是可能是习惯了造轮子，对于框架的使用总是觉得有些别扭，有种说不出的感觉。
​		本次项目链端基于fisco-bcos，使用的是python的sdk，前端使用Vue.js的框架并且搭配上ElementUI，后端使用python-flask框架。其实整个过程总结下来并不难：
​		首先是智能合约的编写，solidity总体上来说与java差不多，所以学起来还是比较快的。但是要注意pure、view、storage、memory之类的使用。然后理清楚本次项目中需要实现四个基本功能的逻辑代码即可。	
​		然后，要从console.py中明白提供的区块链相关的api是如何被使用的（主要是call、sendtx、deploy的使用），我们的参数是如何被读入的。一开始call这个api的参数就让我有点疑惑，其中有个参数为【abi文件】就卡了我很久，但事实上把他理解为一个json文件会简单许多，它储存合约中函数、公有变量的参数、类型等一系列信息，对于其读入其实就类似json的读入。
​		之后书写后端代码，基于RESTful的api来写会很方便，并且可以使用swagger为我们自动生成后端代码的脚手架。之后从console.py模仿api的调用。输入的参数来源于前端，而api的的调用则算是后端与链端进行交互了。借此，前端、后端、链端就联系起来了。
​		此处不赘述前端、后端的学习，但是做这个项目，自己学习前后端的搭建也花费了不少的时间，最终做出来的时候其实还是挺有成就感的，但是由于临近期末，多个课程都有大作业，所以其实时间还是蛮紧张的，做的时候压力也是蛮大的，有种赶鸭子上架的感觉。