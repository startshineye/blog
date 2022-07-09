###  web前端框架设计
  我们一般基于python做web开发的话一般使用的是django的web开发框架。
  基于Django的汽车知识图谱web前端框架设计  
我们首先了解下Django：
#### Django开发框架设计
https://www.djangoproject.com/

#### Django交互流程
   ![](../images/52.png) 
   我们的一般响应式的结果如下:
   ![](../images/53.png) 
   基于传统的MVC的架构设计方式：
模型：跟数据库打交道的。
视图：响应web前端的交互式数据。
模版：就是前端展示给大家可以看到的内容，一般指代前端的页面。

   访问人员请求controller控制器，然后控制器解析用户的请求，获取模型数据（模型数据是从数据库中获取），然后
将模型数据返回给视图(View);视图拿到这个数据之后将数据返回给用户。  
   
##### 1、Django路由
   Django的路由规则如下所示：  
   
```renderscript
"""kgcar URL Configuration

The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/2.0/topics/http/urls/
Examples:
Function views
    1. Add an import:  from my_app import views
    2. Add a URL to urlpatterns:  path('', views.home, name='home')
Class-based views
    1. Add an import:  from other_app.views import Home
    2. Add a URL to urlpatterns:  path('', Home.as_view(), name='home')
Including another URLconf
    1. Import the include() function: from django.urls import include, path
    2. Add a URL to urlpatterns:  path('blog/', include('blog.urls'))
"""

from django.contrib import admin
from django.urls import path
from django.conf.urls import url


from . import index_view
from . import ner_view
from . import relation_view


urlpatterns = [
    path('admin/', admin.site.urls),
    url(r'^$', index_view.index),
    url(r'^ner-post', ner_view.ner_post),
    url(r'^search_entity', relation_view.search_entity),
    url(r'^search_relation', relation_view.search_relation),
]
```

   上面是把用户请求的url转发到对应的控制器.方法上，比如上面将不加任何后缀的请求会转发到:index_view下的index方法。

命名实体查询的请求:^search_entity-->转发到了:relation_view视图的search_entity方法。
实体识别请求:^ner-post-->转发到了：ner_view视图的ner_post方法。
关系查询的请求:^search_relation-->转发到了:relation_view视图的search_relation方法。  

其中:ner_view/relation_view/relation_view指代的是对应的python文件。  

##### 2、Django模版
   路由到对应的controller之后会返回对应的模型然后渲染的视图给模版:template。  
   我们以命名实体识别为例:

路由为:
```buildoutcfg
url(r'^ner-post', ner_view.ner_post)
```


视图为:

```buildoutcfg
# -*- coding: utf-8 -*-
from django.shortcuts import render
from django.views.decorators import csrf
 
import sys
sys.path.append("..")
from toolkit.pre_load import thuFactory,neo4jconn,domain_ner_dict
from toolkit.nlp_ner import get_ner,tempword,get_detail_ner_info,get_ner_info

#中文分词+词性标注+命名实体识别
def ner_post(request):
	ctx ={}
	if request.POST:
		#获取输入文本
		key = request.POST['user_text']
		thu1 = thuFactory
		#中文分词:提前移除空格
		key = key.strip()
		TagList = thu1.cut(key, text=False)
		text = ""
		#命名实体识别
		ner_list = get_ner(key)
		#遍历输出
		for pair in ner_list:
			if pair[1] == 0:
				text += pair[0]
				continue
			if tempword(pair[1]):
				text += "<a href='#'  data-original-title='" + get_ner_info(pair[1]) + "(暂无资料)'  data-placement='top' data-trigger='hover' data-content='"+get_detail_ner_info(pair[1])+"' class='popovers'>" + pair[0] + "</a>"
				continue
			
			text += "<a href='detail.html?title=" + pair[0] + "'  data-original-title='" + get_ner_info(pair[1]) + "'  data-placement='top' data-trigger='hover' data-content='"+get_detail_ner_info(pair[1])+"' class='popovers'>" + pair[0] + "</a>"
		
		ctx['rlt'] = text
			
		seg_word = ""
		length = len(TagList)
		#设置显示格式
		for t in TagList:
			seg_word += t[0]+" <strong><small>["+t[1]+"]</small></strong> "
		seg_word += ""
		ctx['seg_word'] = seg_word

	return render(request, "index.html", ctx)
```

模型为:
```buildoutcfg
# -*- coding: utf-8 -*-
from py2neo import Graph,Node,Relationship,NodeMatcher
#版本说明：Py2neo v4
class Neo4j_Handle():
	graph = None
	matcher = None
	def __init__(self):
		print("Neo4j Init ...")

	def connectDB(self):
		self.graph = Graph("bolt: // localhost:7687", username="neo4j", password="zhangzl")
		self.matcher = NodeMatcher(self.graph)

	#实体查询，用于命名实体识别：品牌+车系+车型
	def matchEntityItem(self,value):
		answer = self.graph.run("MATCH (entity1) WHERE entity1.name = \"" + value + "\" RETURN entity1").data()
		return answer

	#实体查询
	def getEntityRelationbyEntity(self,value):
		#查询实体：不考虑实体类型，只考虑关系方向
		answer = self.graph.run("MATCH (entity1) - [rel] -> (entity2)  WHERE entity1.name = \"" + value + "\" RETURN rel,entity2").data()
		if(len(answer) == 0):
			#查询实体：不考虑关系方向
			answer = self.graph.run("MATCH (entity1) - [rel] - (entity2)  WHERE entity1.name = \"" + value + " \" RETURN rel,entity2").data()
		print(answer)
		return answer

	#关系查询:实体1
	def findRelationByEntity1(self,entity1):
		#基于品牌查询
		answer = self.graph.run("MATCH (n1:Bank {name:\""+entity1+"\"})- [rel] -> (n2) RETURN n1,rel,n2" ).data()
		#基于车系查询，注意此处额外的空格
		if(len(answer) == 0):
			answer = self.graph.run("MATCH (n1:Serise {name:\""+entity1+" \"})- [rel] - (n2) RETURN n1,rel,n2" ).data()
		return answer

	#关系查询：实体2
	def findRelationByEntity2(self,entity1):
		#基于品牌
		answer = self.graph.run("MATCH (n1)<- [rel] - (n2:Bank {name:\""+entity1+"\"}) RETURN n1,rel,n2" ).data()
		if(len(answer) == 0):
			#基于车系
			answer = self.graph.run("MATCH (n1) - [rel] - (n2:Serise {name:\""+entity1+" \"}) RETURN n1,rel,n2" ).data()
		return answer

	#关系查询：实体1+关系
	def findOtherEntities(self,entity,relation):
		answer = self.graph.run("MATCH (n1:Bank {name:\"" + entity + "\"})- [rel:Subtype {type:\""+relation+"\"}] -> (n2) RETURN n1,rel,n2" ).data()
		return answer

	#关系查询：关系+实体2
	def findOtherEntities2(self,entity,relation):
		print("findOtherEntities2==")
		print(entity,relation)
		answer = self.graph.run("MATCH (n1)- [rel:RELATION {type:\""+relation+"\"}] -> (n2:Bank {name:\"" + entity + "\"}) RETURN n1,rel,n2" ).data()
		if(len(answer) == 0):
			answer = self.graph.run("MATCH (n1)- [rel:RELATION {type:\""+relation+"\"}] -> (n2:Serise {name:\"" + entity + " \"}) RETURN n1,rel,n2" ).data()
		return answer

	#关系查询：实体1+实体2(注意Entity2的空格）
	def findRelationByEntities(self,entity1,entity2):
		#品牌 + 品牌
		answer = self.graph.run("MATCH (n1:Bank {name:\"" + entity1 + "\"})- [rel] -> (n2:Bank{name:\""+entity2+" \"}) RETURN n1,rel,n2" ).data()
		if(len(answer) == 0):
			#品牌 + 系列
			answer = self.graph.run("MATCH (n1:Bank {name:\"" + entity1 + "\"})- [rel] -> (n2:Serise{name:\""+entity2+" \"}) RETURN n1,rel,n2" ).data()
		if(len(answer) == 0):
			#系列 + 品牌
			answer = self.graph.run("MATCH (n1:Serise {name:\"" + entity1 + "\"})- [rel] -> (n2:Bank{name:\""+entity2+" \"}) RETURN n1,rel,n2" ).data()
		if(len(answer) == 0):
			#系列 + 系列
			answer = self.graph.run("MATCH (n1:Serise {name:\"" + entity1 + "\"})- [rel] -> (n2:Serise{name:\""+entity2+" \"}) RETURN n1,rel,n2" ).data()
		return answer

	#查询数据库中是否有对应的实体-关系匹配
	def findEntityRelation(self,entity1,relation,entity2):
		answer = self.graph.run("MATCH (n1:Bank {name:\"" + entity1 + "\"})- [rel:subbank {type:\""+relation+"\"}] -> (n2:Bank{name:\""+entity2+"\"}) RETURN n1,rel,n2" ).data()
		if(len(answer) == 0):
			answer = self.graph.run("MATCH (n1:Bank {name:\"" + entity1 + "\"})- [rel:subbank {type:\""+relation+"\"}] -> (n2:Serise{name:\""+entity2+"\"}) RETURN n1,rel,n2" ).data()
		if(len(answer) == 0):
			answer = self.graph.run("MATCH (n1:Serise {name:\"" + entity1 + "\"})- [rel:subbank {type:\""+relation+"\"}] -> (n2:Bank{name:\""+entity2+"\"}) RETURN n1,rel,n2" ).data()
		if(len(answer) == 0):
			answer = self.graph.run("MATCH (n1:Serise {name:\"" + entity1 + "\"})- [rel:subbank {type:\""+relation+"\"}] -> (n2:Serise{name:\""+entity2+"\"}) RETURN n1,rel,n2" ).data()

		return answer
```

模版为：

```buildoutcfg
{% extends "navigate.html" %} {% block mainbody %}

<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
    <title></title>
    <meta charset="utf-8" />
    <script src="/static/js/echarts.common.min.js"></script>   
    <script type="text/javascript" src="http://echarts.baidu.com/gallery/vendors/echarts/echarts-all-3.js"></script>
</head>
<title>实体查询</title>
<div class="container">
    <div class="row">
    <!--head start-->
    <div class="col-md-12">
        <h3 class="page-header"><i class="fa fa-share-alt" aria-hidden="true"></i> 实体查询 </h3>
            <ol class="breadcrumb">
                <li><i class="fa fa-home"></i><a href="\">Home</a></li>
                <li><i class="fa fa-share-alt" aria-hidden="true"></i>实体查询</li>
            </ol>
    </div>
    <div class = "col-md-12">
        <div class="panel panel-default ">
            <header class = "panel-heading">
                查询条件：
            </header>
            <div class = "panel-body">
                <!--搜索框-->
                <form method = "get" id = 'searchEntityForm'>
                    <div >
                        <div class="input-group">
                            <input type="text" id = "user_text" name = "user_text" class="form-control" placeholder="输入实体名称" aria-describedby="basic-addon1">
                            <span class="btn btn-primary input-group-addon" type="button" id="relationSearchButton" style="background-color:#4592fe ; padding:6px 38px" onclick="document.getElementById('searchEntityForm').submit();">查询</span>
                         </div>
                    </div>
                </form>
            </div>
        </div>

        
    </div>
    <p>
        <div class = "col-md-12">
            {% if ctx %}
                <div class="panel panel-default">
                    <header class ="panel-heading">
                        <div class = "panel-body">
                            <h2>数据库中暂未添加该实体</h2>
                        </div>
                    </header>
                </div>
            {% endif %}
        </div>
    </p>
<!--实体关系查询-->
{% if entityRelation %}
    <!-- Echart Dom对象（实体关系） -->
    <div class = "col-md-12">
        <div class="panel panel-default ">
            <header class="panel-heading">
                关系图 :
            </header>
            <div class = "panel-body ">
                <div id="graph" style="width: 90%;height:600px;"></div>
            </div>
        </div>
    </div>
{% endif %}
{% if entityRelation %}
<div class = "col-md-12">
    <div class="panel panel-default">
    <header class="panel-heading">
        关系列表 :
    </header>
        <div class = "panel-body">
            <table class = "table" data-paging =  "true" data-sorting="true"></table>
        </div>
    </div>
</div>
{% endif %}
</div>
</div>
{% if entityRelation %}
<script src="/static/js/jquery.min.js"></script>
<script type="text/javascript">
        // 基于查询结果：初始化Data和Links列表，用于Echarts可视化输出
        var ctx = [ {{ ctx|safe }} ] ;
        //{entity2,rel}
        var entityRelation = [ {{ entityRelation|safe }} ] ;
        var data = [] ;
        var links = [] ;
        if(ctx.length == 0){
            var node = {} ;
            var url = decodeURI(location.search) ;
            var str = "";
            if(url.indexOf("?") != -1){
                str = url.split("=")[1]
            }
            //实体1：待查询的对象
            node['name'] = str;
            node['draggable'] = true ;
            var id = 0; 
            node['id'] = id.toString() ;
            data.push(node) ;

            //实体2：查询并转储到data中，取二者较小的值
            var maxDisPlayNode = 25 ;
            for( var i = 0 ;i < Math.min(maxDisPlayNode,entityRelation[0].length) ; i++ ){
                node = {} ;
                node['name'] = entityRelation[0][i]['entity2']['name'] ;
                node['draggable'] = true ;   //是否允许拖拽
                //是否URL：区分类型
                if('url' in entityRelation[0][i]['entity2']){
                    node['category'] = 1 ;
                }
                else{
                    node['category'] = 2 ;
                }
                id = i + 1
                node['id'] = id.toString();
                var flag = 1 ;
                relationTarget = id.toString() ;
                for(var j = 0 ; j<data.length ;j++){
                    if(data[j]['name'] === node['name']){
                        flag = 0 ;
                        relationTarget = data[j]['id']  ;
                        break ;
                    }
                }
                relation = {}
                relation['source'] = 0 ;
                relation['target'] = relationTarget ;
                relation['category'] = 0 ;               

                if(flag === 1){
                    data.push(node) ;
                    relation['value'] = entityRelation[0][i]['rel']['type'] ;
                    relation['symbolSize'] = 10
                    links.push(relation) ;
                }
                else{
                    maxDisPlayNode += 1 ;
                    for(var j = 0; j<links.length ;j++){
                        if(links[j]['target'] === relationTarget){
                            links[j]['value'] = links[j]['value']+" | "+entityRelation[0][i]['rel']['type'] 
                            break ;
                        }
                    }

                }
            }
            //基于表格的展示++
            tableData = []
            for (var i = 0 ; i < entityRelation[0].length ; i++){
                relationData = {} ;
                relationData['entity1'] = str ;
                relationData['relation'] = entityRelation[0][i]['rel']['type'] ;
                relationData['entity2'] = entityRelation[0][i]['entity2']['name'] ;
                tableData.push(relationData) ;
            }
            jQuery(function(){
                $('.table').footable({
                "columns": [{"name":"entity1",title:"Entity1"} ,
                          {"name":"relation",title:"Relation"},
                          {"name":"entity2",title:"Entity2"}],
                "rows": tableData
                });
            });
            //基于表格的展示--

        }
        // 基于准备好的数据：Data和Links，设置Echarts参数
        var myChart = echarts.init(document.getElementById('graph'));
        option = {
            title: {
                text: ''
            },                //标题
            tooltip: {},                           //提示框配置
            animationDurationUpdate: 1500,
            animationEasingUpdate: 'quinticInOut',
            label: {
                normal: {
                    show: true,
                    textStyle: {
                        fontSize: 12
                    },
                }
            },                          //节点上的标签
            legend: {
                x: "center",
                show: false
            },
            series: [
                {
                    type: 'graph',                //系列：
                    layout: 'force',
                    symbolSize: 45,
                    focusNodeAdjacency: true,
                    roam: true,
                    edgeSymbol: ['none', 'arrow'],
                    categories: [{
                        name: 'Bank',
                        itemStyle: {
                            normal: {
                                color: "#009800",
                            }
                        }
                    }, {
                        name: 'Serise',
                        itemStyle: {
                            normal: {
                                color: "#4592FF",
                            }
                        }
                    }, {
                        name: 'Instance',
                        itemStyle: {
                            normal: {
                                color: "#C71585",
                            }
                        }
                    }],
                    label: {
                        normal: {
                            show: true,
                            textStyle: {
                                fontSize: 12,
                            },
                        }
                    },               //节点标签样式
                    force: {
                        repulsion: 1000
                    },
                    edgeSymbolSize: [4, 50],
                    edgeLabel: {
                        normal: {
                            show: true,
                            textStyle: {
                                fontSize: 10
                            },
                            formatter: "{c}"
                        }
                    },           //边标签样式
                    data: data,                 //节点
                    links: links,               //节点间的关系
                    lineStyle: {
                        normal: {
                            opacity: 0.9,
                            width: 1.3,
                            curveness: 0,
                            color:"#262626"
                        }
                    }            // 连接线的风格
                }
            ]
        };
        myChart.setOption(option);
</script>
{% endif %}

{% endblock %}
```

##### 3、系统目录结构
   ![](../images/54.png) 

kgcar:是我们构建的工程
model:与数据库交互的文件
static:是我们通用的样式.
template:模版.
toolkit:我们设置的工具.

##### 4、创建项目
   使用 django-admin 来创建 HelloWorld 项目：
   
```renderscript
django-admin startproject HelloWorld
```

创建完成后我们可以查看下项目的目录结构：

```renderscript
$ cd HelloWorld/
$ tree
.
|-- HelloWorld
|   |-- __init__.py
|   |-- asgi.py
|   |-- settings.py
|   |-- urls.py
|   `-- wsgi.py
`-- manage.py
```
   
目录说明：

HelloWorld: 项目的容器。
manage.py: 一个实用的命令行工具，可让你以各种方式与该 Django 项目进行交互。
HelloWorld/__init__.py: 一个空文件，告诉 Python 该目录是一个 Python 包。
HelloWorld/asgi.py: 一个 ASGI 兼容的 Web 服务器的入口，以便运行你的项目。
HelloWorld/settings.py: 该 Django 项目的设置/配置。
HelloWorld/urls.py: 该 Django 项目的 URL 声明; 一份由 Django 驱动的网站"目录"。
HelloWorld/wsgi.py: 一个 WSGI 兼容的 Web 服务器的入口，以便运行你的项目。

接下来我们进入 HelloWorld 目录输入以下命令，启动服务器：

```renderscript
python3 manage.py runserver 0.0.0.0:8000
```



  

 
