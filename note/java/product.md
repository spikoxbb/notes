# listSpuInfo

## 入参

- List<Long> spuIds （商品ID list）
-  String countryCode（国家码）

## 出参

Map<Long, SpuDTO>

## 原接口

productService.getSimpleSpuCountryInfoNotActive

> 原来会默认返回某个语言下标题和图片和标签，这次需要返回所有语言信息

## DTO

```java
public class SpuDTO {

    /**
     * 商品ID
     */
    private Long productId;

    /**
     * 标题
     */
    private Map<String, String> title;

     /**
     * 商品类型
     */
    private Integer productType;

    /**
     * 图片
     */
    private Map<String, String> image;

    /**
     * 国家码
     */
    private String countryCode;

    /**
     * 销量
     */
    private Integer saleCount;

    /**
     * 标签
     */
    private List<Integer> labels;

}
```

# listSkuInfo

## 入参

- List<Long> skuIds （skuID list）
-  String countryCode（国家码）

## 出参

Map<Long, SkuDTO>

## 原接口

productService.getSimpleSkuCountryInfo

## DTO

```java
public class SkuDTO {

    /**
     * 商品ID
     */
    private long spuId;

    /**
     * 商品skuID
     */
    private long skuId;

    /**
     * 商品sku code
     */
    private String skuCode;

    /**
     * 国家码
     */
    private String countryCode;

    /**
     * 划线原价格
     */
    private BigDecimal price;

    /**
     * vip价格
     */
    private BigDecimal priceVip;

    /**
     * 活动建议价
     * */
    private BigDecimal adviseActivityPrice;

    /**
     * 佣金率
     */
    private BigDecimal commissionRate;

    private Map<String, List<SkuAttributeDTO>> attributeMap;

}
```

```java
public class SkuAttributeDTO {

    private Long attrId;

    private String attrName;

    private Long valueId;

    private String valueName;

}
```

# listSkuBySpuIds

## 入参

- List<Long> spuIds （spuID list）
-  String countryCode（国家码）

## 出参

Map<Long, SpuInfoDTO> 

## 原接口

- productService.getSkuInfoByProductId
- productService.getSimpleSpuCountryInfoNotActive

## DTO

```java
public class SpuInfoDTO {

    private SpuDTO spuDTO;

    private List<SkuDTO> skuDTOS;

}
```

