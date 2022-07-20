---
title: Spring) MyBatis mapping
excerpt: resultMap, association, collection
---

# 객체 안의 1:N 관계, 리스트 데이터 가져오기
```
public class CustomProduct {
	private int customProductNo;
	private User user;
	private Product product;
	private int purchaseNo;
	private int count;
	private int price;
	private String cartStatus;
	private Date regDate;
	
	// customParts
	private List<CustomParts> minusParts; //제외재료 
	private List<CustomParts> plusParts; //추가재료 
}
```
```
public class Purchase {
	private int purchaseNo;
	private User user;
	private int price;
	private String address;
	private String name;
	private String phone;
	private String email;
	private String message;
	private String purchaseStatus;
	private String status;
	private Date regDate;
	private String paymentCondition;
	private String imp_uid;
	private int amount;
	private int usePoint;
	
	private List<CustomProduct> customProduct; //커스터마이징상품
}
```
### Assocation (1:N)
- CustomProduct has `User`, `Product`   
- Purchase has `User`  

### Collection (list)
**User  \>  Purchase  \>  CustomProduct  \>  CustomParts(minusParts, plusParts)**   
한명의 유저는 구매내역 여러개  
하나의 구매에는 여러개의 커스터마이징상품  
하나의 커스터마이징상품에는 여러개의 제외재료, 추가재료
- CustomProduct  
`List<CustomParts> minusParts`   
`List<CustomParts> plusParts`  
- Purchase  
`List<CustomProduct> customProduct` <br/><br/>

## Mybatis resultMap
- constructor - id - result - association - collection -discriminator 순으로 작성해야함 <br/><br/>

## MyBatis resultMap association
```
<resultMap id="customProductSelectMap" type="customProduct">
  <association property="user"    javaType="user" >
      <result property="userId"        column="user_id"          jdbcType="VARCHAR"/>
  </association>

  <association property="product" javaType="product" >
      <result property="productNo"     column="product_no"       jdbcType="NUMERIC"/>
      <result property="name"          column="name"             jdbcType="VARCHAR"/>
      <result property="thumbnail"     column="thumbnail"        jdbcType="VARCHAR"/>
      <result property="price"         column="product_price"    jdbcType="NUMERIC"/>
  </association>
</resultMap>
```
<br/><br/>


## MyBatis resultMap collection  
- \<collection property=“” column=“서브쿼리에 들어갈 파라미터” javaType=“java.util.ArrayList” ofType=“” select=“서브쿼리의 id”/\>
- 파라미터가 2개이상이면 : column=“{id=id, name=name}” parameterType=“java.util.Map”

```
<resultMap id="customProductSelectMap" type="customProduct">
    <collection property="minusParts" column="customproduct_no" javaType="java.util.ArrayList" ofType="parts" select="getListMinusParts">
    </collection>
    <collection property="plusParts" column="customproduct_no" javaType="java.util.ArrayList" ofType="parts" select="getListPlusParts">
    </collection>
</resultMap>
```
=> CustomProduct 클래스의 minusParts변수는 “getListMinusParts” 라는 id의 \<statement\>가 실행된 결과값 저장,  
customproduct_no는 getListMinusParts를 실행할 때의 파라미터 인자로 사용
```
<resultMap id="purchaseSelectMap" type="purchase">
    <collection property="customProduct" column="purchase_no" javaType="java.util.ArrayList" ofType="customProduct" select="getListCustomProductByPurchaseNo">
    </collection>
</resultMap>
```
<br/><br/>


## 최종 
```
<resultMap id="customProductSelectMap" type="customProduct">
  <result property="customProductNo" column="customproduct_no" 	jdbcType="NUMERIC"/>
  <result property="count" 	     column="count" 	        jdbcType="NUMERIC"/>
  <result property="price" 	     column="price" 	        jdbcType="NUMERIC"/>
  
  <association property="user"    javaType="user" >
      <result property="userId"   column="user_id" 		jdbcType="VARCHAR"/>
  </association>
  
  <association property="product" javaType="product" >
    <result property="productNo"     column="product_no"       jdbcType="NUMERIC"/>
    <result property="name"          column="name"             jdbcType="VARCHAR"/>
    <result property="thumbnail"     column="thumbnail"        jdbcType="VARCHAR"/>
    <result property="price"         column="product_price"    jdbcType="NUMERIC"/>
  </association>
  
  <collection property="minusParts" column="customproduct_no" javaType="java.util.ArrayList" 
    ofType="parts" select="getListMinusParts">
  </collection>	
  
  <collection property="plusParts" column="customproduct_no" javaType="java.util.ArrayList" 
    ofType="parts" select="getListPlusParts">
  </collection> 
  
</resultMap>
```

```
<resultMap id="purchaseSelectMap" type="purchase">
  <result property="purchaseNo"      column="purchase_no"    jdbcType="NUMERIC"/> 
  <result property="price"           column="price"          jdbcType="NUMERIC"/> 
  <result property="address"         column="address"        jdbcType="VARCHAR"/> 
  <result property="imp_uid"         column="imp_uid"        jdbcType="VARCHAR"/> 
  <result property="amount"          column="amount"         jdbcType="NUMERIC"/> 
  <result property="usePoint"        column="usepoint"       jdbcType="VARCHAR"/>
  
  <association property="user" javaType="user" >
    <result property="userId"        column="user_id"        jdbcType="VARCHAR"/>
    <result property="userName"      column="user_name"      jdbcType="VARCHAR"/>
    <result property="phone"         column="phone"          jdbcType="VARCHAR"/>
    <result property="role"          column="role"           jdbcType="VARCHAR"/>
  </association>
  
  <collection property="customProduct" column="purchase_no" javaType="java.util.ArrayList" 
        ofType="customProduct" select="getListCustomProductByPurchaseNo">
  </collection>
  
</resultMap>
```
<br/>
