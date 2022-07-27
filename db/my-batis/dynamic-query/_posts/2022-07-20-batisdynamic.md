---
title: Spring) MyBatis Dynamic SQL
excerpt: for each, where
---

## MyBatis insert foreach
### collection 
파라미터 타입으로 **넘어온 map 안에**  list 형태로 담을 list의 변수명  
### item 
collection 을 사용**할** 변수명 
### separator 
반복 문자열을 구분할 문자  

```
<insert id="insertMinusParts" parameterType="map">
  INSERT INTO customparts(customparts_no, productparts_no, minus_name, customproduct_no)
  SELECT seq_customparts_no.NEXTVAL, inner_table.*
  FROM(
      <foreach collection="minusParts" item="item" separator="union all">
        SELECT
        #{item.minusNo:NUMERIC} as productparts_no,
        #{item.minusName:VARCHAR} as minus_name,
        #{item.customProductNo:NUMERIC} as customproduct_no
        FROM dual
      </foreach>
    )inner_table
</insert>
``` 

## MyBatis where, if
```
<select id="getSaleTotalCount" resultType="int" parameterType="String">
  SELECT COUNT(*)
    FROM( SELECT *
    FROM purchase
    <where>
        purchase_status!='0'
        
        <if test="searchCondition != null">
          AND purchase_status=#{searchCondition:VARCHAR}
        </if> 
        
     </where>) countTable	
</select>
```
- \<where\> 문 안에 `purchase_status!='0'` 가 없다면 조건이 AND 로 시작함  
  -> WHERE 로 치환해줌  
  -> 뭐가 먼저 들어올지 모르는 매개변수 값에 \<where\> 써주면 됨 <br/>
  
