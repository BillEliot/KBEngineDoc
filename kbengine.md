# KBEngine

> 架构

![](./arch.png)

> Loginapp

* 与客户端的第一个连接点
* 固定端口
* 初始通信时加密
  + 公用密钥对(任意长度的密钥)
  + 用户名 / 密码
* 使用多个Loginapp负载均衡(DNS轮流调度)

> Baseapp

* 与客户端通信的固定点
* 客户端与Cellapp通信的中介
* 与客户端的连接均衡分担在各Baseapp间
* 用于处理没有空间位置属性的Entity
  + 拍卖行
  + 公会管理器
  + 房间匹配管理器
* 每个Baseapp同时担任其它Baseapp容错的角色
* 通常一个CPU/核上处理一个Baseapp

> Base Entity

Baseapp上存在两种实体:
* Entity
* Proxy

## Entity

一般意义上的游戏Entity, 如玩家、NPC、拍卖行, 工会管理器等

## Proxy

* 与客户端连接
* C++继承自KBEngine.Entity
* 特殊的Entity

> Baseapp 容灾处理

* 备份entity到其它Baseapp
![](./dt_1.png)
* Baseapp crash后变得不可用
![](./dt_2.png)
* 灾难发生后快速切换到其他备份的Baseapp
![](./dt_3.png)
* 与Crash的Baseapp连接的客户端会被断开连接
  + 所有的数据都已被存储
  + 当重新连接后，它们将继续与其原来的Entity连接

> Baseapp Manager

* 管理Baseapp间的负载平衡
* 监视所有的Baseapp以实现各个Baseapp之间的容错
* 玩家登录分配和创建Entity
* 一个服务器群组有一个BaseappMgr实例

> Cellapp

* 空间与位置数据的处理
  + 处理玩家交互的Space(空间、房间、场景等)
* 处理Space内的Entity
* 处理Space内的一个域(一个Cellapp在一个Space上的Cell只会有一个, 通常进程占用一个CPU/核, 多个Cell没有意义)
* 一个Cellapp有可能处理多个Space
* 通常一个CPU/核上处理一个Cellapp

> Cellapp 主要负载

* 管理的Entity的总数量
* Entity的通信的频率
  + 用户所调用的方法
  + 系统自动更新的属性
  + Entity的密集度
* Entity脚本
* Entity数据大小

> Entity & Cell

* 每个space至少有一个Entity(通常第一个Entity是SpaceEntity，用于让用户操控Space)
* Cellapp上的每个Player Entity都有一个Witness对象(Witness监视周围的Entity，将发生的事件消息同步到客户端)
* Entity的兴趣范围(AOI)缺省是500M(可自定义的，依赖于很多因素)

> Entity: Real & Ghost

* Real Entity是权威Entity
* Ghost Entity是从邻近的Cell对应的Entity的部分数据的拷贝
![](./real&ghost.png)

> Ghost Entity

**解决跨越cell边界的Entity的交互问题**

* Ghost属性为Read Only, 要更改属性值只能通过方法调用来更新其对应的Real Entity

## 方法调用

转发给其Real Entity

## 属性

属性可以是real only, 例如: 将永远不会存在于ghost上; 如果属性对于客户端是可见的, 那么该属性必须是可以ghost的, 例如：当前的武器、等级、名称等

> Cellapp Manager

* CellappMgr管理:
  + 所有的Cellapp及它们的负载
  + 所有的Cell边界
  + 所有Space
* 管理Cellapp的负载平衡(告诉Cellapps它们的Cell边界应该在哪里)
* 把新建的Entity加入到正确的Cell上
* 一个服务器群组有一个CellappMgr实例

> DB Manager

* 管理Entity数据的存储
* 负责数据库与服务器间Entity的通信
* 支持的数据库类型:
  + MySQL
  + MongoDB
  + Redis
  + Custom DB
* 最好独立机器运行

> Entity 备份

* Baseapp间轮流调度处理
* Baseapp向Cellapp要Entity的Cell部分的数据再定时转给DBMgr存储

> Daemon(进程守护)

* 监视服务器进程
* 每个服务器有一个machine
* 启动/停止服务器进程
* 通知服务器群组各个进程的存活状态
* 监视机器状态(CPU、内存、带宽)

> 登录过程

* 客户端发送登录请求(指定IP、端口)
* Loginapp收到登录请求并解密请求消息(一些客户端也会选择不加密通讯)
* Loginapp转发登录消息到DBMgr
* DBMgr验证用户名、密码(查询数据库)
* 转发请求到BaseappMgr
* BaseappMgr发送创建Player Entity的消息到负载最小的Baseapp
* Baseapp创建一个新的Proxy
* Proxy的TCP端口被返回给客户端

> 实现 Entity

* 每个Entity必须
  + 在`entities.xml`中定义
  + 必须有`<Entity_name>.def`文件
  + 必须有`<Entity_name>.py`文件
* 每个Entity可以
  + 有最多3个部分的实现 (Client/Cell/Base)
  + 使用common路径下的共享的脚本

> 分布式 Entity

![](./entity_1.png)

> Entity 基本定义

`<entity_name>.def`

```xml
<root>
    <Properties>
    </Properties>

    <ClientMethods>
    </ClientMethods>

    <BaseMethods>
    </BaseMethods>

    <CellMethods>
    </CellMethods>
</root>
```

> Entity 继承

**`<entity_name>.def` 中定义继承**

## 两种继承机制

* `<Parent>`
  + 继承所有的东西
  + 属性、方法
  + Volatile 属性
  + LOD 级别
  + 简单级别的继承
* `<Implements>`
  + 继承属性和方法
  + 多级别的继承

**Example - Avatar Entity**

```xml
<root>
    <Volatile>
        <position />
        <!--<position> 0 </position> Don't update-->
        <yaw />
        <pitch />
        <roll />
    </Volatile>

    <Implements>
        <Interface>GameObject</Interface>
        <Interface>State</Interface>	
    </Implements>
    <Properties>
        <roleType>
            <Type>UINT8</Type>
            <Flags>BASE</Flags>
            <Default>0</Default>
            <Persistent>true</Persistent>
        </roleType>
    </Properties>

    <BaseMethods>
        <createCell>
            <Arg>ENTITYCALL</Arg>
        </createCell>
    </BaseMethods>

    <CellMethods>
        <jump>
            <Exposed />
        </jump>
    </CellMethods>

    <ClientMethods>
        <onJump />
    </ClientMethods>
</root>
```

> Entity 属性

* Type(网络传输、数据库存储标准化)
* Utype(固定协议ID)         
* Default(缺省值)
* Flags(属性发布标志)
* Detail Level(LOD)
* Volatile
* Persistent(是否数据持久化)

## Cell 上的属性

* Entity数据被频繁访问
* 跨越Cell Boundary时数据会被复制(到新的Cell)
* 数据备份到Base
* 数据改变时通知客户端:
  + 属性的改变
  + 当一个entity进入玩家的AOI时

## Base 上的属性

* 更复杂
* 访问较少
* 数据改变时通知客户端

## Client 上的属性

* 可访问部分server属性
* 属性值从Cell上发布得来
* Cell属性改变会触发`set_<property>()`

> Entity 数据类型

* 简单类型
  + INT8
  + UINT8
  + FLOAT32
  + FLOAT64
  + STRING
  + VECTOR3
  + ...
* 序列类型
  + ARRAY
  + TUPLE

```xml
<root>
   <Properties>
      <name>
        <Type>STRING</Type>
      </name>

      <items>
        <Type>ARRAY
            <of>UINT8</of>
        </Type>
      </items>
   </Properties>
</root>
```

## 复杂类型

* FIXED_DICT
  + Dictionary型对象
  + 固定key集
* PYTHON(比FIXED_DICT低效)
  + 支持任何Python数据类型
  + 安全性(读取客户端传来的数据来序列化Python对象)
  + 使用Python的pickle模块

```xml
<root>
   <Properties>
      <CharacterInfos>
         <Type>FIXED_DICT
            <Properties>
               <name>
                  <Type>STRING</Type>
               </name>
               <level>
                  <Type>UINT8</Type>
               </level>
            </Properties>
         </Type>
      </CharacterInfos>
    </Properties>
</root>
```

**重用的类型定义在<assets>/scripts/entity_defs/types.xml**

> Entity 属性发布(Flag)

## BASE

属于Base
只有Base可以访问
       Baseapp2和Baseapp3无法访问到Baseapp1中红色实体的BASE属性
BASE属性的修改不会被广播
把它们定义在.def文件里就意味着它们会被定期的备份到其它Baseapp上和数据库里

