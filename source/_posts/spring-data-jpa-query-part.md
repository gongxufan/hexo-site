---
layout: post
title: "spring-data-jpa只查询实体部分字段的情况"
date: 2017-03-29 11:46
tags: [spring,java]
category: 随笔
description: 使用jpa查询默认会返回表的全部字段，为了查询效率和安全考虑我们有时候需要控制查询返回的字段范围。

---
不论是nativequery还是hql的query，都可以指定需要查询的字段，只是必须定义这些字段所对应的实体，而且需要一个构造函数，构造函数的参数就是查询的字段列表。举个栗子：
```java
@Entity  
@Table(name="tcHuman")  
@JsonInclude(JsonInclude.Include.NON_NULL)  
public class Human {  
    @Id  
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "tableKeyGenerator")  
    @TableGenerator(name = "tableKeyGenerator", table = "tcTableKeyGenerator",  
            pkColumnName = "pk_key", valueColumnName = "pk_value", pkColumnValue = "humanID",  
            initialValue = 1, allocationSize = 1)  
    private Integer humanID;  
  
    private String humanName;  
    //人员代码  
    private String humanCode;  
    private String humanPassword;  
    private String description;  
    //所属单位  
    private Integer unitID;  
    private Integer displayOrder;  
    private Integer identifyType;  
    private Integer activeFlag;  
  
    public Human() {  
    }  
  
    public Human(String humanName) {  
        this.humanName = humanName;  
    }  
  
    public Integer getHumanID() {  
        return humanID;  
    }  
  
    public void setHumanID(Integer humanID) {  
        this.humanID = humanID;  
    }  
  
    public String getHumanName() {  
        return humanName;  
    }  
  
    public void setHumanName(String humanName) {  
        this.humanName = humanName;  
    }  
  
    public String getHumanCode() {  
        return humanCode;  
    }  
  
    public void setHumanCode(String humanCode) {  
        this.humanCode = humanCode;  
    }  
  
    public String getHumanPassword() {  
        return humanPassword;  
    }  
  
    public void setHumanPassword(String humanPassword) {  
        this.humanPassword = humanPassword;  
    }  
  
    public String getDescription() {  
        return description;  
    }  
  
    public void setDescription(String description) {  
        this.description = description;  
    }  
  
    public Integer getUnitID() {  
        return unitID;  
    }  
  
    public void setUnitID(Integer unitID) {  
        this.unitID = unitID;  
    }  
  
    public Integer getDisplayOrder() {  
        return displayOrder;  
    }  
  
    public void setDisplayOrder(Integer displayOrder) {  
        this.displayOrder = displayOrder;  
    }  
  
    public Integer getIdentifyType() {  
        return identifyType;  
    }  
  
    public void setIdentifyType(Integer identifyType) {  
        this.identifyType = identifyType;  
    }  
  
    public Integer getActiveFlag() {  
        return activeFlag;  
    }  
  
    public void setActiveFlag(Integer activeFlag) {  
        this.activeFlag = activeFlag;  
    }  
}  
```
查询人员的实体，我需要返回所有人员的名字。查询如下：
```java
@Query("select new Human (h.humanName) from Human  h ")  
   List<Human> getHumanList();  
```
我们需要查询humanName,所以必须要有public Human(String humanName)这个构造函数，而且必须提供默认的构造函数,否则entity无法构造。

这样虽然方便，但是如果返回的字段偏多，那么这个构造函数就参数列表就很长。这种情况最好还是用nativequery,定义的接口其返回类型为List<Object[]>。

对于Object[]返回结果的处理如果想做成通用的，可以参考下GSON的反序列化，通过传递T.class，通过反射去构造对象并设置字段。