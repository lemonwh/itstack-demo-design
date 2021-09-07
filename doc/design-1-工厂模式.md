### 工厂方法模式

#### 1.项目简介

| 工程 | 描述                                         |
| ---- | -------------------------------------------- |
| 1-00 | 场景模拟工厂，用于提供三组不同奖品的发放接口 |
| 1-01 | 使用一坨代码实现业务需求，也是对ifelse的使用 |
| 1-02 | 通过设计模式优化改造代码，产生对比性从而学习 |

#### 2.业务解析

| 序号 | 类型               | 接口                                                         |
| ---- | ------------------ | ------------------------------------------------------------ |
| 1    | 优惠券             | CouponResult sendCoupon(String uId, String couponNumber, String uuid) |
| 2    | 实物商品           | Boolean deliverGoods(DeliverReq req)                         |
| 3    | 第三方爱奇艺兑换卡 | void grantToken(String bindMobileNumber, String cardId)      |

#### 3.工厂模式的代码实现

##### 3.1定义发奖接口

```java
public interface ICommodity {

    void sendCommodity(String uId, String commodityId, String bizId, Map<String, String> extMap) throws Exception;

}
```

- 所有的奖品无论是实物、虚拟还是第三方，都需要通过我们的程序实现此接口进⾏行处理，以保证最 终入参出参的统一性。
- 接口的入参包括; 用户ID 、 奖品ID 、 业务ID 以及 扩展字段 用于处理发放实物商品时的收货地址。

##### 3.2实现奖品发放接口

**优惠券**

```java
public class CouponCommodityService implements ICommodity {

    private Logger logger = LoggerFactory.getLogger(CouponCommodityService.class);

    private CouponService couponService = new CouponService();

    public void sendCommodity(String uId, String commodityId, String bizId, Map<String, String> extMap) throws Exception {
        CouponResult couponResult = couponService.sendCoupon(uId, commodityId, bizId);
        logger.info("请求参数[优惠券] => uId：{} commodityId：{} bizId：{} extMap：{}", uId, commodityId, bizId, JSON.toJSON(extMap));
        logger.info("测试结果[优惠券]：{}", JSON.toJSON(couponResult));
        if (!"0000".equals(couponResult.getCode())) throw new RuntimeException(couponResult.getInfo());
    }
}
```

**实物奖品**

```java
public class GoodsCommodityService implements ICommodity {

    private Logger logger = LoggerFactory.getLogger(GoodsCommodityService.class);

    private GoodsService goodsService = new GoodsService();

    public void sendCommodity(String uId, String commodityId, String bizId, Map<String, String> extMap) throws Exception {
        DeliverReq deliverReq = new DeliverReq();
        deliverReq.setUserName(queryUserName(uId));
        deliverReq.setUserPhone(queryUserPhoneNumber(uId));
        deliverReq.setSku(commodityId);
        deliverReq.setOrderId(bizId);
        deliverReq.setConsigneeUserName(extMap.get("consigneeUserName"));
        deliverReq.setConsigneeUserPhone(extMap.get("consigneeUserPhone"));
        deliverReq.setConsigneeUserAddress(extMap.get("consigneeUserAddress"));

        Boolean isSuccess = goodsService.deliverGoods(deliverReq);

        logger.info("请求参数[优惠券] => uId：{} commodityId：{} bizId：{} extMap：{}", uId, commodityId, bizId, JSON.toJSON(extMap));
        logger.info("测试结果[优惠券]：{}", isSuccess);

        if (!isSuccess) throw new RuntimeException("实物商品发放失败");
    }

    private String queryUserName(String uId) {
        return "花花";
    }

    private String queryUserPhoneNumber(String uId) {
        return "15200101232";
    }

}
```

**第三方兑换卡**

```java
public class CardCommodityService implements ICommodity {

    private Logger logger = LoggerFactory.getLogger(CardCommodityService.class);

    // 模拟注入
    private IQiYiCardService iQiYiCardService = new IQiYiCardService();

    public void sendCommodity(String uId, String commodityId, String bizId, Map<String, String> extMap) throws Exception {
        String mobile = queryUserMobile(uId);
        iQiYiCardService.grantToken(mobile, bizId);
        logger.info("请求参数[爱奇艺兑换卡] => uId：{} commodityId：{} bizId：{} extMap：{}", uId, commodityId, bizId, JSON.toJSON(extMap));
        logger.info("测试结果[爱奇艺兑换卡]：success");
    }

    private String queryUserMobile(String uId) {
        return "15200101232";
    }
}
```

- 这⾥我们定义了一个商店的工厂类，在里⾯按照类型实现各种商品的服务。可以⾮常干净整洁的处理你的代码，后续新增的商品在这里扩展即可。如果你不喜欢 if 判断，也可以使用 switch 或者 map 配置结构，会让代码更加干净。
- 另外很多代码检查软件和编码要求，不喜欢if语句后面不写扩展，这里是为了更加⼲净的向你体现 逻辑。在实际的业务编码中可以添加括号。

**3.3测试类**

```java
public class ApiTest {

    @Test
    public void test_commodity() throws Exception {
        StoreFactory storeFactory = new StoreFactory();

        // 1. 优惠券
        ICommodity commodityService_1 = storeFactory.getCommodityService(1);
        commodityService_1.sendCommodity("10001", "EGM1023938910232121323432", "791098764902132", null);

        // 2. 实物商品
        ICommodity commodityService_2 = storeFactory.getCommodityService(2);
        Map<String,String> extMap = new HashMap<String,String>();
        extMap.put("consigneeUserName", "谢飞机");
        extMap.put("consigneeUserPhone", "15200292123");
        extMap.put("consigneeUserAddress", "吉林省.长春市.双阳区.XX街道.檀溪苑小区.#18-2109");

        commodityService_2.sendCommodity("10001","9820198721311","1023000020112221113",new HashMap<String, String>() {{
            put("consigneeUserName", "谢飞机");
            put("consigneeUserPhone", "15200292123");
            put("consigneeUserAddress", "吉林省.长春市.双阳区.XX街道.檀溪苑小区.#18-2109");
        }});

        // 3. 第三方兑换卡(爱奇艺)
        ICommodity commodityService_3 = storeFactory.getCommodityService(3);
        commodityService_3.sendCommodity("10001","AQY1xjkUodl8LO975GdfrYUio",null,null);

    }

}
```

- 另外从运行测试结果上也可以看出来，在进行封装后可以⾮常清晰的看到一整套发放奖品服务的完整性，统⼀了入参、统一了结果。

#### 4.总结

**好处**

避免创建者与具体的产品逻辑耦合、满足单一职责，每一个业务逻辑实现都在所属自己类中完成、满足开闭原则，无需更改使用调用方就可以在程序中引入新的产品类型。

**缺点**

如果有非常多的奖品类型，那么实现的子类会极速扩张。因此需要使用其他的模式进行优化。