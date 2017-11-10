---
layout: post
title: "Javascript树形结构的数组表示和转换"
date: 2016-12-08 17:05
tags: javascript
category: 随笔
description: 树的数据结构数组表示，一种是带pid形式的一维数组，另一种为带子节点的数组表示。

---
在做微信公众号开发的时候，对自定义菜单进行管理就涉及到父子菜单的操作。我们在数据中一般保存为(id,pid,...)这样的平级结构，而在调用微信接口的时候一般是一个层级的数组结构，我们可以用JS进行转换。

## 数组降级
```javascript
/**
 * 将一个层次的数据结构，转换为平行的父子数据结构
 * @param nodesArray
 * @returns {*}
 */
function array2Nodes(nodesArray) {
    var i, l,
        key = "id",
        parentKey = "pId",
        childKey = "sub_button";
    if (!key || key == "" || !nodesArray) return [];

    if (nodesArray instanceof Array) {
        var r = [];
        var tmpMap = {};
        for (i = 0, l = nodesArray.length; i < l; i++) {
            tmpMap[nodesArray[i][key]] = nodesArray[i];
        }
        for (i = 0, l = nodesArray.length; i < l; i++) {
            if (tmpMap[nodesArray[i][parentKey]] && nodesArray[i][key] != nodesArray[i][parentKey]) {
                if (!tmpMap[nodesArray[i][parentKey]][childKey])
                    tmpMap[nodesArray[i][parentKey]][childKey] = [];
                tmpMap[nodesArray[i][parentKey]][childKey].push(nodesArray[i]);
            } else {
                r.push(nodesArray[i]);
            }
        }
        return r;
    } else {
        return [nodesArray];
    }
}
```
array2Nodes将带有子节点的数据展开，变成平级的一维数组

## 维度提升
```javascript
/**
 * 将平行父子数据结构转为层次结构
 * @param nodes
 * @returns {Array}
 */
function nodes2Array(nodes) {
    if (!nodes) return [];
    var childKey = "sub_button",
        r = [];
    if (nodes instanceof Array) {
        for (var i = 0, l = nodes.length; i < l; i++) {
            r.push(nodes[i]);
            if (nodes[i][childKey])
                r = r.concat(nodes2Array(nodes[i][childKey]));
        }
    } else {
        r.push(nodes);
        if (nodes[childKey])
            r = r.concat(nodes2Array(nodes[childKey]));
    }
    return r;
}
```
nodes2Array按照指定的key生成自包含的数组，这里是微信的sub_button(其他场景可以自定义key)。

##　测试
```javascript
var menuArray = [{
    "id": "AE9CB92989FFB1523D8C63C3124AB372F074F8898BE7B930FABCE06BF60CB237",
    "pId": "-1",
    "key": "4C0F11D31571F75A00F620C82E5847E89837FDD758C5027E4D1DDFE959697F85",
    "url": "http://",
    "media_id": "null",
    "name": "菜单名称",
    "type": "view",
    "orderId": 1,
    "msgText": "null"
}, {
    "id": "FD5C1B24A580DB7DE7F472736B7A06277FE8C17BB8739F9D3BE2B19D866C8470",
    "pId": "-1",
    "key": "909CE8ADA256BEEE2E37A78C0CDD823B273B04C60F797A537EC174A0377D7B8C",
    "url": "null",
    "media_id": "null",
    "name": "菜单名称",
    "type": "click",
    "orderId": 2,
    "msgText": "null"
}, {
    "id": "324703D728232820DE9A8305D183B3289AE708DBBF4B8F4D044F7BC9714B7F80",
    "pId": "FD5C1B24A580DB7DE7F472736B7A06277FE8C17BB8739F9D3BE2B19D866C8470",
    "key": "C9B73D24A5EAD91772CFEC5BD129B7EB2C2A9A96E3EF743BAF54B53E29758837",
    "url": "null",
    "media_id": "null",
    "name": "菜单名称",
    "type": "click",
    "orderId": 1,
    "msgText": "null"
}, {
    "id": "00DB54295F5A87248BEE3B7160C217066A6CE77C00B9EC2A415CE15898390425",
    "pId": "AE9CB92989FFB1523D8C63C3124AB372F074F8898BE7B930FABCE06BF60CB237",
    "key": "05810B96BBDC28AAB99EBC69E70F9196C42434A042C0468F6A8DDB0875D22972",
    "url": "null",
    "media_id": "null",
    "name": "菜单名称",
    "type": "click",
    "orderId": 1,
    "msgText": "null"
}, {
    "id": "547B7E126D694B4DC9B831951B3F5A2A378ED4AA0DA9955C088A076F865BC1B1",
    "pId": "AE9CB92989FFB1523D8C63C3124AB372F074F8898BE7B930FABCE06BF60CB237",
    "key": "4EA0242BE5E66E90FB7931B804472DE68CE17D46817CD699F38E21903B1DEAE6",
    "url": "null",
    "media_id": "null",
    "name": "菜单名称",
    "type": "click",
    "orderId": 2,
    "msgText": "null"
}];

var nodes = array2Nodes(menuArray);
console.log(array2Nodes(nodes));
console.log(nodes2Array(nodes));
```
在控制台的输出如下：
![img](/upload/images/javascript-tree-data/result.png)

## 应用
在后台管理系统中树的操作是很频繁的，一般在后台查询采用递归，而这样往往逻辑复杂不易维护。比如要展示一颗部门的树，我们先定义个Unit类：
```java
/**
 * Created by gongxufan on 2017/3/13.
 */
@Entity
@Table(name="tcUnit")
public class Unit {

    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "tableKeyGenerator")
    @TableGenerator(name = "tableKeyGenerator", table = "tcTableKeyGenerator",
            pkColumnName = "pk_key", valueColumnName = "pk_value", pkColumnValue = "unitID",
            initialValue = 1, allocationSize = 1)
    private Integer unitID;
    private Integer seniorUnitID;
    private String unitName;
    private String description;
    private Integer displayOrder;

    public Integer getUnitID() {
        return unitID;
    }

    public void setUnitID(Integer unitID) {
        this.unitID = unitID;
    }

    public Integer getSeniorUnitID() {
        return seniorUnitID;
    }

    public void setSeniorUnitID(Integer seniorUnitID) {
        this.seniorUnitID = seniorUnitID;
    }

    public String getUnitName() {
        return unitName;
    }

    public void setUnitName(String unitName) {
        this.unitName = unitName;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public Integer getDisplayOrder() {
        return displayOrder;
    }

    public void setDisplayOrder(Integer displayOrder) {
        this.displayOrder = displayOrder;
    }
}
```
里边的注解大可不必在意，那是spring-data-jpa的一些注解。这是经典的自包含结构，一个unitID再来一个 seniorUnitID在指向父节点。这个是业务节点，然后咱们定义一个easyUI树所需的节点数据结构。
```java
public class TreeNode {
    private String id;
    private String  pId;
    private String text;
    private boolean checked;
    private String state;
    private String iconCls;
    private int type;

    private Map<String,Object> attributes;

    public Map<String, Object> getAttributes() {
        return attributes;
    }

    public void setAttributes(Map<String, Object> attributes) {
        this.attributes = attributes;
    }

    public void setType(int type) {
        this.type = type;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getpId() {
        return pId;
    }

    public void setpId(String pId) {
        this.pId = pId;
    }

    public int getType() {
        return type;
    }

    public String getText() {
        return text;
    }

    public void setText(String text) {
        this.text = text;
    }

    public boolean isChecked() {
        return checked;
    }

    public void setChecked(boolean checked) {
        this.checked = checked;
    }

    public String getState() {
        return state;
    }

    public void setState(String state) {
        this.state = state;
    }

    public String getIconCls() {
        return iconCls;
    }

    public void setIconCls(String iconCls) {
        this.iconCls = iconCls;
    }
```
里边包含id,pId以及树节点的基本信息，显然前端展示所需的数据需要从Unit结构进行一次转换。现在我们需要展示所有单位的树形结构，那么我们只需要查询所有的Unit记录保存到List，然后将这个List转换为TreeNode类型即可：
```java
    @ResponseBody
    @RequestMapping(value = "/unit/getUnitTree",method = RequestMethod.POST)
    public Object getUnitTree(Integer humanId,Integer fetchHumans, Integer fetchWhole,HttpServletRequest request){
        if(humanId == null){
            humanId = EgovaWebUtil.getHumanInSession(request).getHumanID();
        }
        List<TreeNode> treeNodes = new ArrayList<TreeNode>();
        ResultDto<List<Unit>> resultDto =  humanAction.getHumanUnitTree(humanId,fetchWhole);

        if(ApiInvokeUtil.validateAPiInvoke(resultDto)){
            List<Unit> unitList = resultDto.getData();
            //将单位列表转换为easyUI树形结构所需的格式
            if(unitList != null && unitList.size() > 0){
                for(int i = 0 ; i < unitList.size();i++){
                    TreeNode treeNode = new TreeNode();
                    Unit unit = unitList.get(i);
                    treeNode.setText(unit.getUnitName());
                    treeNode.setChecked(false);
                    treeNode.setId("unit:" + unit.getUnitID());
                    treeNode.setIconCls(TreeNodeConst.NODE_ICON_UNIT);
                    treeNode.setType(TreeNodeConst.NODE_TYPE_UNIT);
                    treeNode.setState("open");
                    treeNode.setpId("unit:" + unit.getSeniorUnitID());
                    Map<String,Object> attrs = ObjectUtil.Object2Map(unit);
                    attrs.put("nodeType",TreeNodeConst.NODE_TYPE_UNIT);
                    treeNode.setAttributes(attrs);
                    treeNodes.add(treeNode);
                    //查询单位下的人员
                    if(fetchHumans == null)
                        dealHumansInUnit(unit,treeNodes);
                }
            }
        }else{
            return resultDto;
        }
        ResultDto<List<TreeNode>> treeNodesDTO = new ResultDto<List<TreeNode>>();
        treeNodesDTO.setData(treeNodes);
        treeNodesDTO.setCode(ErrorCode.OK.getCode());
        treeNodesDTO.setDesc(ErrorCode.OK.getDesc());
        return treeNodesDTO;
    }
```
这里是springmvc的控制器的业务处理逻辑了，这里包含对单位下人员的子节点的构造。不管里面的业务，最终目的就是生成树节点的一个List结构。

后台就干这么个事儿，不用去递归生成一个TreeNode(TreeNode parent)这样的自包含数据。递归结构交给Js去做，前端只需用array2Nodes函数进行转换即可：
```javascript
  $.post(BASEPATH + '/config/unit/getUnitTree',function (response) {
            if(common.dealResponse(response)){
                //转换List为树形结构
                var nodes = common.array2Nodes(response.data);
                //加载部门树
                $('#unit-tree').tree({
                    data : nodes,
                    onClick : function(node){
                        //获取节点对应的目录属性信息
                        var attributes = node.attributes;
                        if (attributes) {
                            //根据单位节点和人员节点显示不同的面板
                            if(attributes.nodeType == common.constant.NODE_TYPE_UNIT){
                                $("#unitPanel").show();
                                $("#humanPanel").hide();
                                $("#unitForm").form('load',attributes);
                            }else{
                                $("#humanPanel").show();
                                $("#unitPanel").hide();
                                $("#humanForm").form('load',attributes);
                            }
                        } else {
                            $("#unitPanel").hide();
                            $("#humanPanel").hide();
                        }
                    },
                    onLoadSuccess: function (node) {
                        var $tree = $(this);
                        var root = $tree.tree('getRoot');
                        var treeNodes = $tree.tree('getChildren');
                        var root = treeNodes[0];
                        $(root.target).trigger('click');
                        // 展开该节点
                        $tree.tree('expand', root.target);
                    }
                });
            }
        });
```
## end
在如今js大前端大行其道加之客户端浏览器性能得到改善的情况，把一些计算任务交给客户端，能减少服务端的访问压力。