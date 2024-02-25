title: solr上手指南

date: 2016-04-23 22:10:59

tags: [solr]

categories: 搜索

---

1. 通过homebrew安装solr:
   ```
   brew install solr
   ```
   安装在`/usr/local/Cellar/solr/`下对应版本号的目录

2. 启动solr cloud:
   ```
   solr start -e cloud -noprompt
   ```
   ~~或者solr单个节点:~~
   ```
   solr start
   ```
   启动后通过访问:http://localhost:8983/solr/#/~cloud 可以看到系统拓扑图:
   ![2016-04-12 at 下午9.20.jpg](/image/B30F4FC6D27D77B47CEF0E3E8241C4E4.jpg)

3. 导入数据
   3.1 导入富文本文件
   ```
   post -c gettingstarted /usr/local/Cellar/solr/5.5.0/libexec/docs/
   ```
   以上命令将导入solr目录下的示例数据文件夹包含的富文本文件.
   导入之后,就可以通过访问solr的搜索界面:http://localhost:8983/solr/#/gettingstarted_shard1_replica2/query 进行搜索:
   ![2016-04-12 at 下午9.34.jpg](/image/62A148F2E36F570A76B83CF1E9AA8998.jpg)
   3.2 通过solr xml格式文件导入数据
   solr xml文件的定义参见:[apache wiki](https://cwiki.apache.org/confluence/display/solr/Uploading+Data+with+Index+Handlers#UploadingDatawithIndexHandlers-XMLFormattedIndexUpdates)
   导入示例文件命令为:
   ```
   post -c gettingstarted example/exampledocs/*.xml
   ```
   通过访问:http://localhost:8983/solr/gettingstarted/browse 可以看到当前已导入的文件列表,同样可以通过搜索界面进行搜索.
   3.3 通过json格式文件导入
   ```
   post -c gettingstarted example/exampledocs/books.json
   ```
   3.4 通过csv格式文件导入
   ```
   post -c gettingstarted example/exampledocs/books.csv
   ```
   3.5 其他导入方式
   通过DIH从数据库导入,使用SolrJ插件通过java等基于jvm的代码导入,使用Admin UI ...
4. 通过关键词进行搜索
   4.1 单个条件搜索
   通过替换搜索界面中q的默认值:`*:*`为'foudation'或者访问地址:http://localhost:8983/solr/gettingstarted/select?wt=json&indent=true&q=foundation 可以得到搜索结果.
   **访问搜索界面需要执行sharding,但是请求搜索结果可以忽略**
   可以看到返回的结果几乎包含了所有的记录,因为刚才的搜索包含了所有的字段,通过将q设置为`name:foudation`可以仅仅搜索`name`属性中中包含foudation的记录,同理可以试试q设置为`cat:software`搜索cat属性中包含software的记录.
   4.2 词组搜索
   例如需要搜索`CAS latency`,在搜索界面中可以直接录入`CAS latency`,但是如果通过url,需要将空格替换为`+`.
   ![2016-04-12 at 下午10.17.jpg](/image/64ACBB392AD1C988DBDBBE74381E0896.jpg)
   4.3 组合搜索条件
   需要条件出现使用`+`,需要条件不出现使用`-`
   例如:http://localhost:8983/solr/gettingstarted/select?wt=json&indent=true&q=%2Bone+%2Bthree 这个链接搜索包含one但是不包含three的记录
   4.4 更多
   访问[apache wiki](https://cwiki.apache.org/confluence/display/solr/Searching)获取更多有关搜索的细节.
5. 清理
   到现在我们已经大致接触了solr,要将solr恢复到初始状态,执行以下命令:
   ```
   bin/solr stop -all ; rm -Rf example/cloud/
   ```






