### 建造者模式

#### 0.前言

一个项目上线往往要经历业务需求，产品设计，研发实现，测试验证，上线部署到正式开量，而这其中对研发非常重要的一块就是研发实现的过程，又可以包括：架构选型，功能设计，设计评审，代码实现，代码评审，单测覆盖率检查，编写文档，提交测试。

#### 1.项目简介

| 工程 | 描述                                                       |
| ---- | ---------------------------------------------------------- |
| 3-00 | 场景模拟工程，模拟装修过程中的套餐选择（豪华，田园，简约） |
| 3-01 | 使用一坨代码实现业务需求，也是对ifelse的使用               |
| 3-02 | 通过设计模式优化改造代码，产生对比性从而学习               |

建造者模式所完成的内容就是通过对多个简单对象通过一步步的组装建出一个复杂对象的过程。

例如你玩王者荣耀的时候，初始化界面有三条路，树木，野怪，守卫塔，甚至依赖于你的网络情况会控制清晰度，而当你换一个场景进行其他不同模式的选择时，同样会建设道路，树木，野怪等等，但是他们的摆放和大小都有不同。这里就可以用到建造者模式来初始化游戏元素。

#### 2.业务解析

- 物料接口提供了基本信息，以保证所有的装修材料都可以统一标准进行获取。
- 装修配置都实现了物料接口，接下来我们会通过案例去使用不同的物料组合出不同的套餐服务。

#### 3.建造者模式的代码实现

建造者模式主要解决的问题在软件系统中，有时候面临“一个复杂对象”的创建工作，其通常由各个部分的子对象用一定的过程构成；由于需求的变化，这个复杂对象的各个部分经常面临重大的变化，但是将他们组合在一起的过程却相对稳定。

这里我们会把构建的过程交给创建者类，而创建者通过使用我们的构建工具包，去构建出不同的装修套装。

| 物料                   | 包装接口（Imenu） | 创建者（Builder） |
| ---------------------- | ----------------- | ----------------- |
| 吊顶、涂料、地板、地砖 | 包装接口实现      | 各种组装的实现    |

- Builder，建造者类具体的各种组装由此类实现。
- DecorationPackageMenu，是IMenu接口的实现类，主要是承载建造过程中的填充器。相当于这是一套承载物料和创建者中间衔接的内容。

##### 3.1定义装修包装接口

```java
public interface IMenu {

    /**
     * 吊顶
     */
    IMenu appendCeiling(Matter matter);

    /**
     * 涂料
     */
    IMenu appendCoat(Matter matter);

    /**
     * 地板
     */
    IMenu appendFloor(Matter matter);

    /**
     * 地砖
     */
    IMenu appendTile(Matter matter);

    /**
     * 明细
     */
    String getDetail();

}
```

- 接口类中定义了填充各项物料的方法：吊顶、涂料、地板、地砖，以及最终提供获取全部明细的方法。

##### 3.2装修包实现

**DecorationPackageMenu**

```java
/**
 * 装修包
 */
public class DecorationPackageMenu implements IMenu {

    private List<Matter> list = new ArrayList<Matter>();  // 装修清单
    private BigDecimal price = BigDecimal.ZERO;      // 装修价格

    private BigDecimal area;  // 面积
    private String grade;     // 装修等级；豪华欧式、轻奢田园、现代简约

    private DecorationPackageMenu() {
    }

    public DecorationPackageMenu(Double area, String grade) {
        this.area = new BigDecimal(area);
        this.grade = grade;
    }

    public IMenu appendCeiling(Matter matter) {
        list.add(matter);
        price = price.add(area.multiply(new BigDecimal("0.2")).multiply(matter.price()));
        return this;
    }

    public IMenu appendCoat(Matter matter) {
        list.add(matter);
        price = price.add(area.multiply(new BigDecimal("1.4")).multiply(matter.price()));
        return this;
    }

    public IMenu appendFloor(Matter matter) {
        list.add(matter);
        price = price.add(area.multiply(matter.price()));
        return this;
    }

    public IMenu appendTile(Matter matter) {
        list.add(matter);
        price = price.add(area.multiply(matter.price()));
        return this;
    }

    public String getDetail() {

        StringBuilder detail = new StringBuilder("\r\n-------------------------------------------------------\r\n" +
                "装修清单" + "\r\n" +
                "套餐等级：" + grade + "\r\n" +
                "套餐价格：" + price.setScale(2, BigDecimal.ROUND_HALF_UP) + " 元\r\n" +
                "房屋面积：" + area.doubleValue() + " 平米\r\n" +
                "材料清单：\r\n");

        for (Matter matter: list) {
            detail.append(matter.scene()).append("：").append(matter.brand()).append("、").append(matter.model()).append("、平米价格：").append(matter.price()).append(" 元。\n");
        }

        return detail.toString();
    }

}
```

- 装修包的实现中每一个方法都返回了this，也就可以非常方便的用于连续填充各项物料。
- 同时在填充时也会根据物料计算平米数下的报价，吊顶和涂料按照平米数适量乘以常数计算，最后同样提供了统一的获取装修清单的明细方法。

```java
public class Builder {
    public IMenu levelOne(Double area) {
        return new DecorationPackageMenu(area, "豪华欧式")
                .appendCeiling(new LevelTwoCeiling())    // 吊顶，二级顶
                .appendCoat(new DuluxCoat())             // 涂料，多乐士
                .appendFloor(new ShengXiangFloor());     // 地板，圣象
    }

    public IMenu levelTwo(Double area){
        return new DecorationPackageMenu(area, "轻奢田园")
                .appendCeiling(new LevelTwoCeiling())   // 吊顶，二级顶
                .appendCoat(new LiBangCoat())           // 涂料，立邦
                .appendTile(new MarcoPoloTile());       // 地砖，马可波罗
    }

    public IMenu levelThree(Double area){
        return new DecorationPackageMenu(area, "现代简约")
                .appendCeiling(new LevelOneCeiling())   // 吊顶，二级顶
                .appendCoat(new LiBangCoat())           // 涂料，立邦
                .appendTile(new DongPengTile());        // 地砖，东鹏
    }
}
```

- 建造者的使用中就已经非常容易了，统一的建造方式，通过不同物料填充出不同的装修风格，如果将来业务扩展也可以将这部分内容配置到数据库自动生成。但整体的思想还可以使用创建者模式进行搭建。

#### 4.测试验证

**编写测试类**

```java
public class ApiTest {
    @Test
    public void test_Builder(){
        Builder builder = new Builder();

        // 豪华欧式
        System.out.println(builder.levelOne(132.52D).getDetail());

        // 轻奢田园
        System.out.println(builder.levelTwo(98.25D).getDetail());

        // 现代简约
        System.out.println(builder.levelThree(85.43D).getDetail());
    }
}
```

- 如果后续有扩展的需求，可以按照这样的类型方式进行补充，同时对于改造上来说，并没有改动原来的方法，降低了修改成本。

#### 5.总结

- 当一些基本物料不会变，而其组合经常变化的时候，就可以选择这样的设计模式来构建代码。
- 这个设计模式满足了单一职责以及可复用的技术、建造者独立、易扩展、便于控制细节风险。但同时出现特别多的物料以及很多的组合后，类的不断扩展也会造成难以维护的问题。但这种设计结构模型可以把重复的内容抽象到数据库中。

