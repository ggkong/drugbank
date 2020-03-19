需求：点击datatable 的一行diaoyon数据，显示一个模态框，模态框显示该行的全部属性

解决方法： 点击datatable 这一行后，首先构造一个模态框 而后利用ajax 将行的某个属性传回后台 后台d调用数据库将属性全部查询后 用json 形式发送给前台 前台的模态框进行渲染：

实现步骤：

1、构建一个模态框：具体使用方法可以在网上找：https://www.w3h5.com/post/74.html

```html
<div class="container">
    <h2>创建模态框（Modal）</h2>
    <!-- 按钮触发模态框 -->
    <button class="btn btn-primary btn-lg" data-toggle="modal" data-target="#myModal">开始演示模态框</button>
    <!-- 模态框（Modal） -->
    <div class="modal fade" id="myModal" tabindex="-1" role="dialog" aria-labelledby="myModalLabel" aria-hidden="true">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <button type="button" class="close" data-dismiss="modal" aria-hidden="true">&times;</button>
                    <h4 class="modal-title" id="myModalLabel">模态框（Modal）标题</h4>
                </div>
                <div class="modal-body">在这里添加一些文本</div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-default" data-dismiss="modal">关闭</button>
                    <button type="button" class="btn btn-primary">提交更改</button>
                </div>
            </div><!-- /.modal-content -->
        </div><!-- /.modal -->
    </div>
</div>
```

2、将表格中的部分数据发送到前端：

```js
在进行页面跳转时 csrf 协议 即ajaxsetup 是必须要加上的
$.ajaxSetup({
            data: {
              csrfmiddlewaretoken: '{{ csrf_token }}'
            },
          });
下面是ajax 主要部分  
url: 将要跳转至的url 这条url 的主要目的是为了执行 view 中的各种操作（这条url 的模式是url->view 而非普通的url->view->templates）
data :将要发送给后端(即url对应的view) 的数据 通过这个数据我们就可以对数据库进行操作
datatype json : 数据是以json形式进行传输的
type: 对后端进行post 请求
success: 前端将数据发送给后端 后端经过数据处理后 发送给前端  即整个ajax 操作成功时候应该做的事情 
          $.ajax({
            url: '/modal_ajax/',
            data: {
              host: table.row('.selected').data()['de_id']
            },
            dataType: "json",
            type: 'post',
            success: function (data) {
              // alert(data['host'])
            }
          })
```

3、构建上一步所说的url

```python
# 2020 3 10 孔格添加 用来 进行模态框的ajax 传值
url('modal_ajax/', rawview.modal_ajax)
```

4、在上一步url 对应 的view 中进行操作

里面需要注意的是 :由于我们上一步是用ajax 进行操作的 传回的请求是post  所以必须有

request.is_ajax()

利用data  = request.POST  得到前端的数据

然后将前端的值进行处理 然后发送给前端  数据类型一定要是json

对于django 中的model 对象其实有多种操作方式将其转化为json  例：

```python
# 将models 转化为 字典
dict_drug = model_to_dict(modal_drug)
# 将字典转化为json 对象
json_drug = serializers.serialize("json", dict_drug)
```


但是不符合需求 所以我用了最笨的方法

```python
# 一定要记得引入json 包
from django.http import JsonResponse
def modal_ajax(request):
    if request.is_ajax():
       # 直接获取所有的post请求数据
       data = request.POST
       # 获取其中的某个键的值
       modal_drug_de_id = request.POST.get("host")
       print(data)
       print('modal de_id get is ----------------------------->{}'.format(modal_drug_de_id))
       _,drug = models.ChildrenAndGrands.check_one_two_three_and_get(de_id = modal_drug_de_id)
       modal_drug_de_id = drug.de_id
       modal_drug_smiles = drug.smiles
       modal_drug_mw = drug.mw
       modal_drug_logp = drug.logp
       modal_drug_h_a = drug.h_a
       modal_drug_h_d = drug.h_d
       modal_drug_r_b = drug.r_b
       modal_drug_tpsa = drug.tpsa
       modal_drug_qed = drug.qed
       modal_drug_sascore = drug.sascore
       modal_drug_parents_id = drug.parents_id
       modal_drug_ns_id = drug.ns_id
       modal_drug_pdb_code = drug.pdb_code
       modal_drug_drugbank_code = drug.drugbank_code
       modal_drug_bindingdb_code = drug.bindingdb_code
       modal_drug_chembl_code = drug.chembl_code
       modal_drug_ccdc_code = drug.ccdc_code

       # 将models 转化为 字典
    #    dict_drug = model_to_dict(modal_drug)
    #    将字典转化为json 对象
    #    json_drug = serializers.serialize("json", dict_drug)
       
       response = JsonResponse({
           'modal_drug_de_id':modal_drug_de_id,
           'modal_drug_mw':modal_drug_mw,
           'modal_drug_smiles': modal_drug_smiles,
           'modal_drug_logp' :  modal_drug_logp,
           'modal_drug_h_a' :  modal_drug_h_a,
           'modal_drug_h_d' : modal_drug_h_d,
           'modal_drug_r_b' : modal_drug_r_b,
           'modal_drug_tpsa' : modal_drug_tpsa,
           'modal_drug_qed' : modal_drug_qed,
           'modal_drug_sascore' : modal_drug_sascore,
           'modal_drug_parents_id' : modal_drug_parents_id,
           'modal_drug_ns_id' : modal_drug_ns_id,
           'modal_drug_pdb_code' : modal_drug_pdb_code,
           'modal_drug_drugbank_code' : modal_drug_drugbank_code,
           'modal_drug_bindingdb_code' : modal_drug_bindingdb_code,
           'modal_drug_chembl_code' : modal_drug_chembl_code,
           'modal_drug_ccdc_code' : modal_drug_ccdc_code,
           })
       return response
    else:
        return render(request,'404.html')
```

