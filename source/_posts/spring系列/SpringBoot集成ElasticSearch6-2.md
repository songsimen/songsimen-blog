---
title: SpringBoot集成ElasticSearch6.2
date: 2018-08-28 14:06:11
tags: ElasticSearch
categories: springboot
---


elasticsearch (Java High Level REST Client) api
-----------------------------------------------
Java高级REST客户端在Java低级REST客户端之上工作。它的主要目标是公开API特定的方法，接受请求对象作为参数并返回响应对象，以便客户端自己处理请求编组和响应非编组。

可以同步或异步调用每个API。同步方法返回响应对象，而名称以async后缀结尾的异步方法需要一旦收到响应或错误就通知（在由低级客户端管理的线程池上）的侦听器参数。

Java高级REST客户端依赖于Elasticsearch核心项目。它接受与the相同的请求参数，TransportClient并返回相同的响应对象。

兼容性
---
Java高级REST客户端需要Java 1.8并依赖于Elasticsearch核心项目。客户端版本与客户端开发的Elasticsearch版本相同。它接受与the相同的请求参数，TransportClient 并返回相同的响应对象

代码初始化方式
-------

``` java
RestHighLevelClient client = new RestHighLevelClient(
        RestClient.builder(
                new HttpHost("localhost", 9200, "http"),
                new HttpHost("localhost", 9201, "http")));
```

IDE新建SpringBoot项目
pom.xml配置

``` java
  <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>6.2.3</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>6.2.3</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>6.2.3</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client-sniffer</artifactId>
            <version>6.2.3</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.16</version>
        </dependency>
```

application.properties
``` bash
server.port=80
server.servlet.context-path=/es-boot
spring.data.elasticsearch.cluster-nodes=192.168.1.191:9200
```
数据配置，进行初始化操作

``` java
/**
 *  elasticsearch spring-data 目前支持的最高版本为5.5 所以需要自己注入生成客户端
 *
 * 数据配置，进行初始化操作
 * @author qinxuewu
 * @version 1.00
 * @time 28/8/2018下午 5:54
 */
@Configuration
public class ESConfiguration implements FactoryBean<RestHighLevelClient>, InitializingBean, DisposableBean {
    private static final Logger LOG = LoggerFactory.getLogger(ESConfiguration.class);

    @Value("${spring.data.elasticsearch.cluster-nodes}")
    private String clusterNodes;

    private RestHighLevelClient restHighLevelClient;

    /**
     * 控制Bean的实例化过程
     *
     * @return
     * @throws Exception
     */
    @Override
    public RestHighLevelClient getObject() throws Exception {
        return restHighLevelClient;
    }
    /**
     * 获取接口返回的实例的class
     *
     * @return
     */
    @Override
    public Class<?> getObjectType() {
        return RestHighLevelClient.class;
    }

    @Override
    public void destroy() throws Exception {
        try {
            if (restHighLevelClient != null) {
                restHighLevelClient.close();
            }
        } catch (final Exception e) {
            LOG.error("Error closing ElasticSearch client: ", e);
        }
    }

    public boolean isSingleton() {
        return false;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        restHighLevelClient = buildClient();
    }

    private RestHighLevelClient buildClient() {
        try {
            restHighLevelClient = new RestHighLevelClient(RestClient.builder(new HttpHost(clusterNodes.split(":")[0], Integer.parseInt(clusterNodes.split(":")[1]), "http")));
        } catch (Exception e) {
            LOG.error(e.getMessage());
        }
        return restHighLevelClient;
    }


}

```

``` java
@Component
public class EsDao {
    private static final Logger LOG = LoggerFactory.getLogger(EsDao.class);
    @Autowired
    private RestHighLevelClient client;


    /**
     * 判断索引是否存在
     * @param index 索引(关系型数据库)
     * @return
     */
    public boolean isIndexExist(String index){
        try {
            GetRequest getRequest=new GetRequest(index);
            getRequest.fetchSourceContext(new FetchSourceContext(false));
            getRequest.storedFields("_none_");
            boolean exists = client.exists(getRequest);
            return exists;
        }catch (Exception e){
            LOG.error("判断索引是否存在是否存在异常",e);
        }
        return false;
    }

    /**
     * 判断索引是否存在
     * @param index  索引(关系型数据库)
     * @param type   类型(关系型数据表)
     * @param id     数据ID
     * @return
     */
    public boolean isIndexExist(String index,String type,String id){
        try {
            GetRequest getRequest=new GetRequest(index,type,id);
            getRequest.fetchSourceContext(new FetchSourceContext(false));
            getRequest.storedFields("_none_");
            boolean exists = client.exists(getRequest);
            return exists;
        }catch (Exception e){
            LOG.error("判断索引是否存在是否存在异常",e);
        }
        return false;
    }

    /**
     * 创建索引
     * @param index  索引(关系型数据库)
     * @param type   类型(关系型数据表)
     * @param obj    数据源
     * @return
     */
    public void createIndexOne(String index, String type, JSONObject obj) {
        try {
            IndexRequest request = new IndexRequest(index, type);
            request.source(obj.toJSONString(), XContentType.JSON);
            client.index(request);

        } catch (Exception e) {
            LOG.error("创建索引异常", e);
        }
    }

        /**
         * 创建索引
         * @param index  索引(关系型数据库)
         * @param type   类型(关系型数据表)
         * @param id     数据ID
         * @param obj    数据源
         * @return
         */
        public void createIndexOne(String index, String type,String id, JSONObject obj){
            try {
                IndexRequest request=new IndexRequest(index,type,id);
                request.source(obj.toJSONString(),XContentType.JSON);
                client.index(request);
            }catch (Exception e){
                LOG.error("创建索引异常",e);
            }

        }

        /**
         * 批量创建索
         * @param index  索引(关系型数据库)
         * @param type   类型(关系型数据表)
         * @param list   数据源
         */
        public void bacthIndex(String index, String type,List<JSONObject> list){
            try {
                List<IndexRequest> requests = new ArrayList<>();
                list.forEach(i->{
                    requests.add(generateNewsRequest(index,type,i));
                });
                BulkRequest bulkRequest = new BulkRequest();
                for (IndexRequest indexRequest : requests) {
                    bulkRequest.add(indexRequest);
                }
                client.bulk(bulkRequest);
            }catch (Exception e){
                LOG.error("批量创建索引异常",e);
            }
        }
        public static IndexRequest generateNewsRequest(String index, String type,JSONObject obj){
            IndexRequest indexRequest = new IndexRequest(index, type);
            indexRequest.source(obj.toJSONString(),XContentType.JSON);
            return indexRequest;
        }

        /**
         * 删除索引
         * @param index  索引(关系型数据库)
         * @param type   类型(关系型数据表)
         * @param id     数据ID
         * @return
         */
        public boolean deleteIndex(String index,String type,String id){
            try {
                DeleteRequest request=new DeleteRequest(index,type,id);
                client.delete(request);
                return true;
            }catch (Exception e){
                LOG.error("删除索引异常",e);
            }
            return  false;
        }

        /**
         * 修改索引
         * @param index  索引(关系型数据库)
         * @param type   类型(关系型数据表)
         * @param id     数据ID
         * @param obj    数据源
         * @return
         */
        public boolean updateIndex(String index, String type,String id, JSONObject obj){
            try {
                UpdateRequest updateRequest = new UpdateRequest(index,type,id);
                updateRequest.doc(obj.toJSONString(),XContentType.JSON);
                client.update(updateRequest);
                return true;
            }catch (Exception e){
                LOG.error("修改索引异常",e);
            }
            return  false;
        }

        /**
         * 查询单条索引
         * @param index  索引(关系型数据库)
         * @param type   类型(关系型数据表)
         * @param id     数据ID
         */
        public GetResponse findById(String index, String type,String id){
            try {
                GetRequest getRequest=new GetRequest(index,type,id);
                GetResponse getResponse = client.get(getRequest);
                return getResponse;
            } catch (Exception e) {
                LOG.error("查询单条索引异常",e);
            }
            return null;
        }

        /**
         * 查询单条索引
         * @param index     索引(关系型数据库)
         * @param type      类型(关系型数据表)
         * @param id        数据ID
         * @param includes  显示字段
         * @param excludes  排除字段
         */
        public GetResponse findById(String index, String type,String id,String [] includes,String [] excludes){
            try {
                GetRequest getRequest=new GetRequest(index,type,id);
                FetchSourceContext fetchSourceContext = new FetchSourceContext(true, includes, excludes);
                getRequest.fetchSourceContext(fetchSourceContext);
                GetResponse getResponse = client.get(getRequest);
                return  getResponse;
            } catch (Exception e) {
                LOG.error("查询单条索引异常",e);
            }
            return null;
        }

        /**
         * 查询列表索引
         * @param index        索引(关系型数据库)
         * @param type         类型(关系型数据表)
         * @return
         */
        public SearchResponse getAllIndex(String index,String type){
            SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
            SearchRequest searchRequest = new SearchRequest(index);
            searchRequest.types(type);
            searchRequest.source(sourceBuilder);
            try {
                SearchResponse response = client.search(searchRequest);
                return response;
            } catch (Exception e) {
                e.printStackTrace();
                LOG.error("查询列表索引异常",e);
            }
            return  null;
        }

    /**
     * 查询列表索引
     * @param index        索引(关系型数据库)
     * @param type         类型(关系型数据表)
     * @param includes     显示字段
     * @param excludes     排除字段
     * @return
     */
    public SearchResponse getAllIndex(String index, String type,String [] includes,String [] excludes){
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.fetchSource(includes,excludes);
        SearchRequest searchRequest = new SearchRequest(index);
        searchRequest.types(type);
        searchRequest.source(sourceBuilder);
        try {
            SearchResponse response = client.search(searchRequest);
            return response;
        } catch (Exception e) {
            e.printStackTrace();
            LOG.error("查询列表索引异常",e);
        }
        return  null;
    }

        /**
         * 查询列表索引
         * @param index        索引(关系型数据库)
         * @param type         类型(关系型数据表)
         * @param sourceBuilder  查询条件
         * @return
         */
        public SearchResponse getAllIndex(String index, String type, SearchSourceBuilder sourceBuilder){
            SearchRequest searchRequest = new SearchRequest(index);
            searchRequest.types(type);
            searchRequest.source(sourceBuilder);
            try {
                SearchResponse response = client.search(searchRequest);
                return response;
            } catch (Exception e) {
                e.printStackTrace();
                LOG.error("查询列表索引异常",e);
            }
            return  null;
        }
}
```

**创建索引**

``` java
@Autowired
private RestHighLevelClient client;
@Autowired
private EsDao esDao;

    @Test
    public void createIndexOne() {
        try {
            String index="testdb";  //必须为小写
            String type="userinfo";
            JSONObject obj=new JSONObject();
            obj.put("name","qxw");
            obj.put("age",25);
            obj.put("sex","男");
            String [] tags={"标签1","标签2"};
            obj.put("tags",tags);
            esDao.createIndexOne(index,type,obj);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```
**批量创建索引**

``` java
    @Test
    public void bacthIndex(){
        String index="testdb";  //必须为小写
        String type="userinfo";
        List<JSONObject>  list=new ArrayList<>();
        JSONObject obj=null;
        for (int i = 0; i <10 ; i++) {
            obj=new JSONObject();
            obj.put("name","qxw"+i);
            obj.put("age",25+i);
            list.add(obj);
        }
        esDao.bacthIndex(index,type,list);
    }
```
**根据ID查询**
``` java
    @Test
    public void findById(){
        String index="testdb";  //必须为小写
        String type="userinfo";
        String id="NWrCg2UBU-HvVB1XZxe1";
        String result=esDao.findById(index,type,id);
        System.out.println("查询结果："+result);
    }
```
**修改操作**

``` java
   @Test
    public void update(){
        String index="testdb";  //必须为小写
        String type="userinfo";
        String id="NWrCg2UBU-HvVB1XZxe1";
        JSONObject obj=new JSONObject();
        obj.put("name","xiaoming");
        obj.put("time","2018-08-29 00:00:00");
        esDao.updateIndex(index,type,id,obj);
    }
```
**根据ID查询 指定过滤字段**
``` java
    /**
     * 根据ID查询
     */
    @Test
    public void findById(){
        String index="testdb";  //必须为小写
        String type="userinfo";
        String id="NWrCg2UBU-HvVB1XZxe1";
        GetResponse res=esDao.findById(index,type,id);
        System.out.println("查询结果index："+res.getIndex());
        System.out.println("查询结果type："+res.getType());
        System.out.println("查询结果id："+res.getId());
        System.out.println("查询结果source："+res.getSource());
    }
  /**
     * 根据ID查询 指定过滤字段
     */
    @Test
    public void findByIdexcludes(){
        String index="testdb";  //必须为小写
        String type="userinfo";
        String id="NWrCg2UBU-HvVB1XZxe1";
        String [] includes={"name","sex","age"};//不过滤
        String [] excludes={"tags"}; //过滤字段

        System.out.println("查询结果："+esDao.findById(index,type,id,includes,excludes));
    }
```
**查询所有**

``` java
 @Test
    public  void  getAllIndex(){
        String index="testdb";  //必须为小写
        String type="userinfo";
        String result=esDao.getAllIndex(index,type);
        System.out.println("查询结果："+result);

        String [] includes={"name","sex",};//不过滤
        String [] excludes={"tags","age"}; //过滤字段
        String result2=esDao.getAllIndex(index,type,includes,excludes);
        System.out.println("指定过滤字段查询结果："+result2);

    }

```
**条件查询 /匹配所有**

``` java
    @Test
    public  void  getAllIndexByFiled(){
        String index="testdb";
        String type="userinfo";
        /**
         * 使用QueryBuilder
         * termQuery("key", obj) 完全匹配
         * termsQuery("key", obj1, obj2..)   一次匹配多个值
         * matchQuery("key", Obj) 单个匹配, field不支持通配符, 前缀具高级特性
         * multiMatchQuery("text", "field1", "field2"..);  匹配多个字段, field有通配符忒行
         * matchAllQuery();         匹配所有文件
         */

        //匹配所有文件
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder.query(QueryBuilders.matchAllQuery());
        String result=esDao.getAllIndex(index,type,searchSourceBuilder);
        System.out.println("匹配所有查询结果："+result);
    }
```
**模糊、排序查询**

``` java
  /**
     * 条件查询
     */
    @Test
    public  void  getAllIndexByFiled3(){
        String index="testdb";
        String type="userinfo";
   
        SearchSourceBuilder search3 = new SearchSourceBuilder();
//        MatchQueryBuilder matchQueryBuilder = new MatchQueryBuilder("name","qxw");
         //在匹配查询上启用模糊匹配
//        matchQueryBuilder.fuzziness(Fuzziness.AUTO);
//        //在匹配查询上设置前缀长度选项
//        matchQueryBuilder.prefixLength(3); 
//        //设置最大扩展选项以控制查询的模糊过程
//        matchQueryBuilder.maxExpansions(10); 
        
        
        //默认情况下，搜索请求会返回文档的内容,设置fasle不会返回窝
//        search3.fetchSource(false);
        
        //也接受一个或多个通配符模式的数组，以控制以更精细的方式包含或排除哪些字段
        String[] includeFields = new String[] {"name", "age", "tags"};
        String[] excludeFields = new String[] {"_type","_index"};
        search3.fetchSource(includeFields, excludeFields);
        
        //指定排序
        search3.sort(new FieldSortBuilder("age").order(SortOrder.DESC));




         //启用模糊查询 fuzziness(Fuzziness.AUTO)
//        search3.query(QueryBuilders.matchQuery("name","qxw").fuzziness(Fuzziness.AUTO));

        //模糊查询，?匹配单个字符，*匹配多个字符
//        search3.query(QueryBuilders.wildcardQuery("name","*qxw*"));

        //搜索name中或tags  中包含有qxw的文档（必须与music一致)
//        search3.query(QueryBuilders.multiMatchQuery("qxw","name","tags"));



        //多条件查询 相当于and
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        //查询age=32
        TermQueryBuilder termQuery=QueryBuilders.termQuery("age",32);

        //匹配多个值  相当于sql 中in(....)操作
        TermsQueryBuilder termQuerys=QueryBuilders.termsQuery("_id","PWrIg2UBU-HvVB1XzRce","XWqYhGUBU-HvVB1Xahct");

        //模糊查询name中包含qxw
        WildcardQueryBuilder queryBuilder = QueryBuilders.wildcardQuery("name", "*qxw*");

        boolQueryBuilder.must(termQuery);
        boolQueryBuilder.must(queryBuilder);
        boolQueryBuilder.must(termQuerys);

//        //设置from确定结果索引的选项以开始搜索。默认为0。
//        search3.from(0);
//        //设置size确定要返回的搜索命中数的选项。默认为10。
//        search3.size(1);

        search3.query(boolQueryBuilder);

        SearchResponse result=esDao.getAllIndex(index,type,search3);
        //解析SearchHits
        SearchHits hits = result.getHits();
        long totalHits = hits.getTotalHits();
        float maxScore = hits.getMaxScore();

        SearchHit[] searchHits = hits.getHits();
        for (SearchHit hit : searchHits) {
            String indexs = hit.getIndex();
            String types = hit.getType();
            String ids = hit.getId();

            String sourceAsString = hit.getSourceAsString();
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            System.out.println("id ："+ids+sourceAsMap.toString());
        }

        System.out.println("查询结果："+esDao.getAllIndex(index,type,search3));
    }

```

聚合操作
----

``` java
 @Test
    public  void AggregationsTest() throws IOException {
          String index="emptydb";
          String  type="empty";
//        List<JSONObject>  list=new ArrayList<>();
//        JSONObject obj=new JSONObject();
//        obj.put("name","小明"); obj.put("age",25); obj.put("salary",10000); obj.put("detpty","技术部");
//        list.add(obj);
//
//        JSONObject obj2=new JSONObject();
//        obj2.put("name","小蛋"); obj2.put("age",22); obj2.put("salary",5000); obj2.put("detpty","技术部");
//        list.add(obj2);
//
//        JSONObject obj3=new JSONObject();
//        obj3.put("name","张三"); obj3.put("age",24); obj3.put("salary",300); obj3.put("detpty","销售部");
//        list.add(obj3);
//
//        JSONObject obj4=new JSONObject();
//        obj4.put("name","李四"); obj4.put("age",22); obj4.put("salary",4000); obj4.put("detpty","采购部");
//        list.add(obj4);
//
//          //添加测试数据
//        esDao.bacthIndex(index,type,list);



        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        SearchRequest searchRequest = new SearchRequest(index);
        searchRequest.types(type);
        searchRequest.source(searchSourceBuilder);

        //计算所有员工的平均年龄
        //terms(查询字段别名).field(分组字段)
        searchSourceBuilder.aggregation(AggregationBuilders.avg("average_age").field("age"));
        SearchResponse res=client.search(searchRequest);
        System.out.println("聚合操作查询结果："+res.toString());


        Aggregations aggregations = res.getAggregations();
        Map<String, Aggregation> aggregationMap = aggregations.getAsMap();
        System.out.println("聚合操作解析："+aggregationMap.toString());
    }
```

![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)