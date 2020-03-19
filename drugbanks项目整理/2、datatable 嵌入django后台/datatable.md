需求：在页面进行一个表格的渲染 并对表格中的一些属性进行排序

解决方法：由于表格内的数据过于庞大 一次渲染肯定要很长很长的时间 而且排序放在前端进行排序也需要很长很长的时间  我们把排序放在后台 并且减少渲染的数据量，仅渲染页面上的字值

具体的操作步骤：

1、在view 层写好datatable 的后台 这里需要注意的是：

这个后台是经过重载的：

代码中的 all_data  为在图表中显示的数据源（数据源经过处理以后为普通list） 即发送到前端的数据源  前端的js也会实时的和后台进行交互 实现数据排序 数据换页等一系列操作

其中的 propeties  是和后台的 js 代码中cloums （列变量）一一进行对应的 在这个需求中主要执行后台排序的工作 ：（工作的步骤：前端点击排序功能并（监听下按照何种属性何种顺序 ） 传输给后台  后台相应到前端 然后根据  order_name = propeties[order_column]  调用all_data = sorted(chain(drugs),key= attrgetter(order_name), reverse = False) 进行排序）

由于是异步渲染 所以肯定不会将要渲染的数据一次性发送到前端的 ，根据all_data[start:start+length] 来进行切片操作（即选择将要渲染的对象）

将要渲染的 对象发送到前端  

```python
class UsersList110Json(BaseDatatableView):
    total_records = -1
    total_display_records = -1
    
    def get_initial_queryset(self):
        print(self.request.GET)
        propeties = {#这个地方写上表格的列序号和真实抬头之间的对应关系，下面好order_by使用名称
            # '0': 'smiles',
            # 将不在需要smiles 进行显示
            '1': 'de_id',
            '2': 'logp',
            '3': 'mw',
            '4': 'h_a',
            '5': 'h_d',
        }
        
        draw = self.request.GET.get('draw')

        # 将要排序的列序号
        order_column = self.request.GET.get('order[0][column]')
        # 将要排序的方式  即按正序还是逆序
        order_dir = self.request.GET.get('order[0][dir]')

        start = int(self.request.GET.get('start'))
        length = int(self.request.GET.get('length'))

        
        de_id = self.request.GET.get('de_id')
        # print('我得到的deid'+ de_id)
        print('Ajax get de_id is ------------------->{}'.format(de_id))

        # 获得 list 对象
        # drugs = models.LevelTwo.objects.filter(parent_id = de_id)
        
        drugs = models.ChildrenAndGrands.get_children_and_grands(de_id = de_id)
        
        order_name = propeties[order_column]
        print('order by --------------->{}'.format(order_name))
        # 默认用 升序
        all_data = sorted(chain(drugs),key= attrgetter(order_name), reverse = False)


        if order_dir == 'desc':
            all_data =  sorted(chain(drugs),key= attrgetter(order_name), reverse=True)
        
        self.total_records = self.total_display_records = len(all_data)
        # 生成将要显示的图片
        # 调用静态方法
        
        starts = datetime.now()
        was_svg = all_data[start:start+length]
        make_ajax_svg(was_svg)
        ends = datetime.now() 
        print('processing svg------lasts: ', (ends - starts).seconds)        
        
        # print('all_data [10]------------------->{}'.format(all_data[10]))
        return all_data[start:start+length]
        # return all_data
    

    def filter_queryset(self, qs):
        return qs
    
    def get_context_data(self, *args, **kwargs):
        try:
            self.initialize(*args, **kwargs)

            # prepare columns data (for DataTables 1.10+)
            self.columns_data = self.extract_datatables_column_data()

            # determine the response type based on the 'data' field passed from JavaScript
            # https://datatables.net/reference/option/columns.data
            # col['data'] can be an integer (return list) or string (return dictionary)
            # we only check for the first column definition here as there is no way to return list and dictionary
            # at once
            self.is_data_list = True
            if self.columns_data:
                self.is_data_list = False
                try:
                    int(self.columns_data[0]['data'])
                    self.is_data_list = True
                except ValueError:
                    pass

            # prepare list of columns to be returned
            self._columns = self.get_columns()

            # prepare initial queryset
            qs = self.get_initial_queryset()

            # prepare output data
            if self.pre_camel_case_notation:
                aaData = self.prepare_results(qs)

                ret = {'sEcho': int(self._querydict.get('sEcho', 0)),
                       'iTotalRecords': self.total_records,
                       'iTotalDisplayRecords': self.total_display_records,
                       'aaData': aaData
                       }
            else:
                data = self.prepare_results(qs)

                ret = {'draw': int(self._querydict.get('draw', 0)),
                       'recordsTotal': self.total_records,
                       'recordsFiltered': self.total_display_records,
                       'data': data
                       }
            return ret
        except Exception as e:
            return self.handle_exception(e)  
```

前端的js代码如下：

js 代码用jquery 写成，也是经过重载的 里面的order  保证是使用什么排序 2 表示用 第三列进行排序 asc 表示降序 

中间的二维数组 中的第一个数组表示 长度的值 第二个中的数据用来显示再前端 的选择框中 如果第二个 数组中的数据变成[''十个,'三十个','五十个'] ,前端的对应位置就会显示为十个 三十个 五十个

colums 表示列变量  表示再在datatable （表格中的显示列）

其中有几个属性：

orderable  表示能否被排序

 searchable 能否被 搜索

 visible  是否被显示在页面中（默认为显示）

最后ajax 部分中的

url  是向后端发送的url 地址（在第二步中会提及  在第三步中会进行 定义） 

data 为向后端发送的数据

```js
// 显示在 tbody 中的内容
$(document).ready(function() {
    var dt_table = $('.datatable').dataTable({
        language: dt_language,  // global variable defined in html
        // 降序还是升序
        order: [[ 2, "asc" ]],
        // 二维数组，第一个数组用来作为长度的值，第二个数组用来作为显示的选项。
        lengthMenu: [[10, 30, 50],['10', '30', '50']],
        // 列 变量
        columns: [
            {
                data: 'imgscr',
                "width": "400px",
                 render: function (data, type, row, meta) {                                           
                     return "<img style = 'width:300px;height:200px;transition:all 1s;' src='" + data + "' />";
                      //还可以给图片加上超链接
                     //return "<a href='" + data + "'>" + data + "</a>";
                },
                orderable: false,
                searchable: true,
                className: "center"
            },

            {
                data: 'de_id',
                orderable: true,
                searchable: true,
                className: "center",
                visible : false,
            },
            {
                data: 'logp',
                orderable: true,
                searchable: true,
                className: "center"
            },
            {
                data: 'mw',
                orderable: true,
                searchable: true,
                className: "center"
            },
            {
                data: 'h_a',
                orderable: true,
                searchable: true,
                className: "center"
            },
            {
                data: 'h_d',
                orderable: true,
                searchable: true,
                className: "center"
            }


        ],

        searching: false,
        processing: true,
        serverSide: true,
        stateSave: false,

        ajax: {
            "url": USERS_LIST_JSON_URL,
            "data": function(d){
                return $.extend( {}, d, {
                    de_id : document.getElementById('de_id').innerText
                });
            }
        }
    });
});

```

2、添加前端的 html代码 用来将list 渲染在前端  这里我只截取了部分代码 具体的代码会在后面贴出

需要注意的是 js 的加载顺序 jQuery frist then bootstrap 最后是额外的js 

下面又一条 var USERS_LIST_JSON_URL = '{% url "users_list_json_110" %}';在上面步骤的ajax 中会进行提到 在下一步中会进行定义

这个指向的是在后台实现的 view 的url  将会在下一步进行创建 最终指向的是  上一步 view 

```html
<script src="{% static 'js/jquery-3.3.1.js' %}"></script>
<script src="{% static 'js/bootstrap.js' %}"></script>
<script src="{% static "loading/ddv_example_1_10.js" %}"></script>
<script type="text/javascript">
    var USERS_LIST_JSON_URL = '{% url "users_list_json_110" %}';
    // translations for datatables
    var dt_language = {
      "emptyTable": "{% trans "No data available in table" %}",
      "info": "{% trans "Showing _START_ to _END_ of _TOTAL_ entries" %}",
      "infoEmpty": "{% trans "Showing 0 to 0 of 0 entries" %}",
      "infoFiltered": "{% trans "(filtered from _MAX_ total entries)" %}",
      "infoPostFix": "",
      "thousands": ",",
      "lengthMenu": "{% trans "Show _MENU_ entries" %}",
      "loadingRecords": "{% trans "Loading..." %}",
      "processing": "{% trans "Processing..." %}",
      "search": "{% trans "Search: " %}",
      "zeroRecords": "{% trans "No matching records found" %}",
      "paginate": {
        "first": "{% trans "First" %}",
        "last": "{% trans "Last" %}",
        "next": "{% trans "Next" %}",
        "previous": "{% trans "Previous" %}"
            },
      "aria": {
        "sortAscending": "{% trans ": activate to sort column ascending" %}",
        "sortDescending": "{% trans ": activate to sort column descending" %}"
            }
    }
</script>

```

3、添加一条url 用来指向 datableview

```python
url(r'^users_data_110/$', UsersList110Json.as_view(), name="users_list_json_110")
```

4、值得一提的是  这个页面在渲染时是分开渲染的

用到了一个url  两个view  最后又渲染到了同一个template

主线为url -->普通view -->template

分线为datatable view----> template

分线的执行过程为 前端提交ajax 请求 匹配相应 的url url 找到 view 进行后台操作 

分线主要执行的任务 ：根据前端的请求周而复始的请求后台操做

5、最后的附件会加上datatable 的一般使用方法

源码在：https://github.com/ggkong/django-datatable--demo

