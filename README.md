### **代码环境**
1. python 3.7
1. pip 19.0.3

---

### **主要引用的第三方库**
1. requests，用于模拟http/https请求
    + 安装： `pip install requests`
    + 文档： [requests中文文档](http://www.python-requests.org/zh_CN/latest/)
1. beautifulsoup4，用于解析网页，得出我们想要的内容。
    + 安装： `pip install beautifulsoup4 `
    + 文档： [bs4中文文档](https://www.crummy.com/software/BeautifulSoup/bs4/doc.zh/)
1. xlwt，将爬到的结果以Excel的形式保存到本地
    + 安装： `pip install xlwt`
    + api： [xlwt api](https://xlwt.readthedocs.io/en/latest/api.html)

---

### **打开网页**
首先打开[boss直聘官网](https://www.zhipin.com/)，选择一个地点，然后输入关键字，点击搜索，这里以深圳、python为例。
![pic](picture1.png)

观察地址栏URL，可以发现有四个参数，分别是query,city,industry和position，query和city很明显是我输入的python和选择的地点深圳；而industry和position也就是公司行业和职位类型，这里没有选择这两项。

---

### **分析网页**
F12打开开发者工具
![](picture2.png)
每一条职位信息都在一个`<li>`标签中，`<li>`标签下的`<div class="job-primary">`就是我们要找的内容，以第一条为例:
```html
<div class="job-primary">
    <div class="info-primary">
        <h3 class="name">
            <a href="/job_detail/3ff9677f5b1e170a1XJ83Nu-FFQ~.html" data-jid="3ff9677f5b1e170a1XJ83Nu-FFQ~" data-itemid="1" data-lid="1wpWFL3VLoV.search" data-jobid="26676346" data-index="0" ka="search_list_1" target="_blank">
                <div class="job-title">Python</div>
                <span class="red">20-40K</span>
                <div class="info-detail"></div>
            </a>
        </h3>
        <p>深圳 南山区 科技园<em class="vline"></em>3-5年<em class="vline"></em>本科</p>
    </div>
    <div class="info-company">
        <div class="company-text">
            <h3 class="name"><a href="/gongsi/2e64a887a110ea9f1nRz.html" ka="search_list_company_1_custompage" target="_blank">腾讯</a></h3>
            <p>互联网<em class="vline"></em>已上市<em class="vline"></em>10000人以上</p>
        </div>
    </div>
    <div class="info-publis">
        <h3 class="name"><img src="https://img2.bosszhipin.com/boss/avatar/avatar_7.png?x-oss-process=image/resize,w_40,limit_0" />王先生<em class="vline"></em>高级开发工程师</h3>
        <p></p>
    </div>
    <a href="javascript:;" data-url="/wapi/zpgeek/friend/add.json?jobId=3ff9677f5b1e170a1XJ83Nu-FFQ~&lid=1wpWFL3VLoV.search" redirect-url="/geek/new/index/chat?id=0abcd66dded613a01HBy39-7GVE~" class="btn btn-startchat">立即沟通
    </a>
</div>
```
这段html代码中可提取的信息：

1. 岗位名称：Python
1. 薪资：20-40k
1. 公司地址：深圳 南山区 科技园
1. 工作经验：3-5年
1. 学历要求：本科
1. 公司名称：腾讯
1. 公司行业：互联网
1. 融资情况：已上市
1. 公司规模：10000人以上

---

### **获取URL**
* 获取城市编码

    url中的city=101280600，显示的是深圳，说明城市名有一个对应的编号，F12 点击Network选中XHR，有一个city.json
![](picture3.png)

```python
# 城市名称转成编码
def get_city_code(city_name):
    response = requests.get("https://www.zhipin.com/wapi/zpCommon/data/city.json")
    contents = json.loads(response.text)
    cities = contents["zpData"]["hotCityList"]
    city_code = contents["zpData"]["locationCity"]["code"]
    for city in cities:
        if city["name"] == city_name:
            city_code = city["code"]
    return city_code
```

* 根据搜索条件，城市编码，行业和职位类型，获取分页的urls

```python
# 获取所有页的url，返回list
def get_urls(query="", city="", industry="", position="", page=1) -> list:
    base_url = "https://www.zhipin.com/job_detail/?query={}&city={}&industry={}&position={}&page={}"
    urls = []
    url = base_url.format(query, city, industry, position, page)
    response = requests.get(url, headers=headers)
    soup = BeautifulSoup(response.text, "lxml")
    page_list = soup.find("div", "page").find_all("a")
    urls.append(url)
    while page_list[len(page_list) - 1]["href"] != "javascript:;":
        page += 1
        url = base_url.format(query, city, industry, position, page)
        urls.append(url)
        response = requests.get(url, headers=headers)
        soup = BeautifulSoup(response.text, "lxml")
        page_list = soup.find("div", "page").find_all("a")
    return urls
```

---

### **获取HTML代码**

```python
def get_html(url):
    response = requests.get(url, headers=headers)
    if responese.status_code == 200:
        return response.text
```

---

### **从HTML中提取信息**
这里要注意一点：岗位信息是在鼠标悬浮在某一条职位信息才会显示出来，同样查看network 选择XHR：
![](picture4.png)
有一个card.json的链接，有jid和lid两个参数
```python
def get_content(html) -> list:
    bs = BeautifulSoup(html, 'lxml')
    contents = []
    for info in bs.find_all("div", "job-primary"):
        job_name = info.find("div", "job-title").get_text()  # 职位名称
        company = info.find("div", "company-text").a.get_text()  # 公司名称
        jid = info.find("div", "info-primary").a["data-jid"]
        lid = info.find("div", "info-primary").a["data-lid"]
        desc = get_job_desc(jid, lid)  # 岗位描述
        texts = [text for text in info.find("div", "info-primary").p.stripped_strings]
        site = texts[0]  # 公司地址
        work_exp = texts[1]  # 工作经验
        edu_bak = texts[2]  # 学历要求
        salary = info.span.get_text()  # 薪资
        companies = [text for text in info.find("div", "company-text").p.stripped_strings]
        industry = companies[0]  # 公司行业
        if len(companies) > 2:
            finance = companies[1]  # 融资情况
            staff_num = companies[2]  # 公司规模
        else:
            finance = None
            staff_num = companies[1]
        contents.append(job_info(job_name, company, industry, finance, staff_num, salary, site, work_exp, edu_bak, desc))
        time.sleep(1)
    return contents
```

---

### **将爬取的数据保存到本地excel**

```python
def save_data(content, city, query):
    file = xlwt.Workbook(encoding="utf-8", style_compression=0)
    sheet = file.add_sheet("job_info", cell_overwrite_ok=True)
    sheet.write(0, 0, "职位名称")
    sheet.write(0, 1, "公司名称")
    sheet.write(0, 2, "行业")
    sheet.write(0, 3, "融资情况")
    sheet.write(0, 4, "公司人数")
    sheet.write(0, 5, "薪资")
    sheet.write(0, 6, "工作地点")
    sheet.write(0, 7, "工作经验")
    sheet.write(0, 8, "学历要求")
    sheet.write(0, 9, "职位描述")
    for i in range(len(content)):
        sheet.write(i+1, 0, content[i]["job_name"])
        sheet.write(i+1, 1, content[i]["company"])
        sheet.write(i+1, 2, content[i]["industry"])
        sheet.write(i+1, 3, content[i]["finance"])
        sheet.write(i+1, 4, content[i]["staff_number"])
        sheet.write(i+1, 5, content[i]["salary"])
        sheet.write(i+1, 6, content[i]["site"])
        sheet.write(i+1, 7, content[i]["work_experience"])
        sheet.write(i+1, 8, content[i]["education_bak"])
        sheet.write(i+1, 9, content[i]["job_desc"])
    file.save(r'c:\projects\{}_{}.xls'.format(city, query))
```


