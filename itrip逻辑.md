# itrip逻辑

## 0. 注解

@Controller

表示作为controller层的类，用于处理浏览器发过来的请求

@Resource

与@Autowired一样，表示下面的对象是从ioc容器里取的

@ApiOperation

提供文档用，编译后有用于展示所有被注解的标准的接口

@ApiImplicitParams

@RequestParam 

请求参数

```
@ApiOperation(value = "用户登录",httpMethod = "POST",
            protocols = "HTTP", produces = "application/json",
            response = Dto.class,notes="根据用户名、密码进行统一认证")	
	@ApiImplicitParams({
		@ApiImplicitParam(paramType="form",required=true,value="用户名",name="name",defaultValue="yao.liu2015@bdqn.cn"),
		@ApiImplicitParam(paramType="form",required=true,value="密码",name="password",defaultValue="111111"),
	})
```

@RequestMapping

路由地址

```
@RequestMapping(value="/dologin",method=RequestMethod.POST,produces= "application/json")
```

## 1. auth模块

授权模块，负责用户的**注册**、**登录**以及token的替换。

### Src:放java代码

### Controller

### LoginController：用户的**登录**与**注销**

#### 用户登录

从前端获取用户名和密码，数据中存的是密码的md5形式，通过调用userService的接口判断用户名密码是不是正确。

* 如果不正确会通过Dto（Data Transfer Object）类会修改ResponseCode并封装ErrorCode，前端根据ErrorCode判断是否成功。
* 如果正确，会根据userAgent和userCode生成token，把token和user一起存在redis里（json字符串方式），这样比较方便，别人拥有了token也可以直接拿到用户，并通过VO（Value Object）返回前端。
  * token生成方式：用UA判断是pc端还是移动端 + userCode的md5加密 + 时间信息 + agent信息
    - why(?)
  * userCode是邮箱或者手机号，注册时会提到

#### 用户注销

* 验证token的有效性
* 从redis中删除token



### TokenController: Token的替换

判断旧token有效无效，如果无效，抛出异常。用旧token去换新token。

如果有效，小于置换保护期（1h）抛出异常，token有过期时间设置，PC端>2h就TIME_OUT过期，不再能够查到数据。如果是手机端，永不过期。

使用jedis完成。



### UserController: 实现手机和邮箱注册

前端获取到用户名和密码，先对用户名进行判断，如果不重复，就用先创建用户（入库）。再邮件激活（spring实现）或者手机号（SDK）激活，激活成功会把激活位置为1，就可以正常开始使用。



### Service

放实现类和接口（功能的实现）



### Exception

放异常类

### Resource：放资源，数据类、各种配置。

## 2.  biz

是主业务模块

### 酒店信息：

```
/**
 * 酒店信息Controller
 * <p/>
 * 包括API接口：
 * 1、根据酒店id查询酒店拓展属性
 * 2、根据酒店id查询酒店介绍，酒店政策，酒店设施
 * 3、根据酒店id查询酒店特色属性列表
 * 4、根据type 和target id 查询酒店图片
 * 5、查询热门城市
 * 6、查询热门商圈列表（搜索-酒店列表）
 * 7、查询数据字典特色父级节点列表（搜索-酒店列表）
 * 8、根据酒店id查询酒店特色、商圈、酒店名称（视频文字描述）
 * <p/>
 * 注：错误码（100201 ——100300）
 * <p/>
 */
```

### 酒店订单系统（全部都要使用token，没有根据地址拦截，所以非常不方便，每次都要从request里调用token）

####  查询订单

利用request获得token，再根据token判断是否失效，如果没有失效就可以根据token得到自己的用户身份，之后在订单页面对订单进行操作。

```
<p>订单类型(orderType)（-1：全部订单 0:旅游订单 1:酒店订单 2：机票订单）：</p>" +
            "<p>订单状态(orderStatus)（0：待支付 1:已取消 2:支付成功 3:已消费 4：已点评）：</p>" +
            "<p>对于页面tab条件：</p>" +
            "<p>全部订单（orderStatus：-1）</p>" +
            "<p>未出行（orderStatus：2）</p>" +
            "<p>待付款（orderStatus：0）</p>" +
            "<p>待评论（orderStatus：3）</p>" +
            "<p>已取消（orderStatus：1）</p>" +
            "<p>成功：success = ‘true’ | 失败：success = ‘false’ 并返回错误码，如下：</p>" +
            "<p>错误码：</p>" +
            "<p>100501 : 请传递参数：orderType </p>" +
            "<p>100502 : 请传递参数：orderStatus </p>" +
            "<p>100503 : 获取个人订单列表错误 </p>" +
            "<p>100000 : token失效，请重登录 </p>")
```

报错信息通过dto返回，比如orderType是空的之类的。

```
if (orderType == null) {
    return DtoUtil.returnFail("请传递参数：orderType", "100501");
}
if (orderStatus == null) {
    return DtoUtil.returnFail("请传递参数：orderStatus", "100502");
}
```

#### 添加订单

难点：判断房间库存（？）

```
<update  id="flushStore" parameterType="java.util.Map" statementType="CALLABLE">
        {call pre_flush_store(
            #{startTime,jdbcType=DATE,mode=IN},
            #{endTime,jdbcType=DATE,mode=IN},
            #{roomId,jdbcType=BIGINT,mode=IN},
            #{hotelId,jdbcType=BIGINT,mode=IN}
          )
        }
    </update >

    <select id="queryRoomStore" resultType="cn.itrip.beans.vo.store.StoreVO"  parameterType="java.util.Map">
              SELECT A.roomId,A.recordDate,A.store from (
              SELECT
                    store.roomId,
                    store.recordDate,
                    DATE_FORMAT(store.recordDate,'%Y-%m-%d'),
	                store.store - (
                            CASE
                            WHEN SUM(ord.count) IS NULL THEN
                                0
                            ELSE
                                SUM(ord.count)
                            END
	                ) AS store
                    FROM
                        itrip_hotel_temp_store store
                    LEFT JOIN itrip_hotel_order ord ON store.roomId = ord.roomId AND ord.orderStatus = 0
                    AND DATE_FORMAT(store.recordDate,'%Y-%m-%d') BETWEEN DATE_FORMAT(ord.checkInDate, '%Y-%m-%d') AND DATE_FORMAT(ord.checkOutDate,'%Y-%m-%d')
                    WHERE   store.roomId = #{roomId}
                    GROUP BY  store.roomId,store.recordDate) AS A
                    WHERE  A.recordDate BETWEEN DATE_FORMAT(#{startTime}, '%Y-%m-%d') AND DATE_FORMAT(#{endTime}, '%Y-%m-%d') ORDER by A.store ASC
    </select>


    <update id="updateRoomStore" parameterType="java.util.Map">
        update itrip_hotel_temp_store set  store=store-#{count}
        WHERE
            roomId=#{roomId}
        AND
            DATE_FORMAT(recordDate, '%Y-%m-%d ')
        BETWEEN
            DATE_FORMAT(#{startTime}, '%Y-%m-%d')
        AND
            DATE_FORMAT(#{endTime}, '%Y-%m-%d')
    </update>
```

也需要获得token

```
"<p>成功：success = ‘true’ | 失败：success = ‘false’ 并返回错误码，如下：</p>" +
"<p>错误码：</p>" +
"<p>100505 : 生成订单失败 </p>" +
"<p>100506 : 不能提交空，请填写订单信息 </p>" +
"<p>100507 : 库存不足 </p>" +
"<p>100000 : token失效，请重登录</p>")
```

订单号的生成方法是：机器码 +日期+（MD5）（商品IDs+毫秒数+1000000的随机数）

```
//生成订单号：机器码 +日期+（MD5）（商品IDs+毫秒数+1000000的随机数）
StringBuilder md5String = new StringBuilder();
md5String.append(itripHotelOrder.getHotelId());
md5String.append(itripHotelOrder.getRoomId());
md5String.append(System.currentTimeMillis());
md5String.append(Math.random() * 1000000);
String md5 = MD5.getMd5(md5String.toString(), 6);

//生成订单编号
StringBuilder orderNo = new StringBuilder();
orderNo.append(systemConfig.getMachineCode());
orderNo.append(DateUtil.format(new Date(), "yyyyMMddHHmmss"));
orderNo.append(md5);
itripHotelOrder.setOrderNo(orderNo.toString());
```

#### 根据信息判断是否有房

需要提供入住时间房间id等信息，先判断是否有房，再提供修改。

#### 支付后查询订单信息

验证token，之后根据订单的id（不是订单号）来查询订单。

#### 利用Logger自动刷新订单

（10分钟和两个小时的区别是啥）顺便讲下.class

```
/***
     * 10分钟执行一次 刷新订单的状态 不对外公布
     */
    //#@Scheduled(cron = "*0 0/10 * * * ?")
private Logger logger = Logger.getLogger(HotelOrderController.class);
public void flushCancelOrderStatus() {
        try {
            boolean flag = itripHotelOrderService.flushOrderStatus(1);
            logger.info(flag ? "刷取订单成功" : "刷单失败");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
 /***
     * 2小时执行一次 刷新订单的状态 不对外公布
     */
    //@Scheduled(cron = "0 0 0/2 * * ?")
    public void flushOrderStatus() {
        try {
            logger.info("刷单程序开始执行.......");
            boolean flag = itripHotelOrderService.flushOrderStatus(2);
            logger.info("刷单程序执行完毕.......");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

```

#### "/scanTradeEnd" 不记得是啥了

### 酒店房间页面

```
/**
 * 酒店房间Controller
 * <p/>
 * 包括API接口：
 * 1、根据入住时间，退房时间等条件，查询酒店房间列表
 * 2、根据type 和target id 查询酒店房间图片
 * 3、查询床型的接口
 * <p/>
 * <p/>
 * 注：错误码（100301 ——100400）
 * <p/>
 */
```

数据库常规查询，无token

* 分页处理

### 评论页面

```
/**
 * 评论Controller
 *
 * 包括API接口：
 * 1、根据type 和target id 查询评论照片
 * 2、据酒店id查询酒店平均分（总体评分、位置评分、设施评分、服务评分、卫生评分）
 * 3、根据酒店id查询评论数量
 * 4、根据评论类型查询评论 分页
 * 5、上传评论图片
 * 6、删除评论图片
 * 7、新增评论信息
 * 8、查看个人评论信息
 * 9、查询出游类型列表
 * 10、新增评论信息页面获取酒店相关信息（酒店名称、酒店图片、酒店星级）
 *
 * 注：错误码（100001 ——100100）
 */
```

* 通过map查询，上传图片删除图片怎么处理

#### 用户信息

```
/**
 * 用户信息Controller
 *
 * 包括API接口：
 * 1、根据UserId、联系人姓名查询常用联系人接口
 * 2、指定用户下添加联系人
 * 3、修改指定用户下的联系人信息
 * 4、删除指定用户下的联系人信息
 *
 * 注：错误码（100401 ——100500）
 *
 */
```

## 3. Search

## 4. trade

## n. Beans

放pojo，**没有任何方法，只存数据的类**：VO DTO + 数据库映射的类

## n.dao

## n.Util

![image-20210307182757044](C:\Users\16778\AppData\Roaming\Typora\typora-user-images\image-20210307182757044.png)

放一些工具，比如很大的数字类，中转英，md5加密，错误码，验证token等，各个模块都可以通过调用Util包来实现需要的功能。