# 爬取智联招聘实训
## 前期知识掌握要求:
熟练掌握:  
requests  
pymysql  
[入门学习](https://9035.gitee.io/spiderintroduce/)

## 目标
爬取智联招聘网数据，并存储到mysql


## 完整代码
```
import requests,time,pymysql

# 连接数据库
conn = pymysql.connect(host='localhost',user='root',password='root',database='zhilianzhaopin_demo',port=3306)
# 连接数据库前准备
cursor = conn.cursor()

sql = """
 insert into zhaopindate(id,job_name,salary,welfare,workingExp,edu,emplType,createDate,company_name,company_size,company_type,company_url,position_url) values (null,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)
"""

zhaopBse = requests.session()


# 职位名称:job_name
# 薪水:salary
# 福利待遇:welfare
# 工作年限:workingExp
# 学历要求：edu
# 全职or兼职:emplType
# 发布时间:createDate
# 公司名称:company_name
# 公司规模:company_size
# 公司类型:company_type
# 公司主页:company_url
# 详情地址:position_url

# 爬取职业
work_page = ["java","python","javascript","liunx","git","nodejs","hadoop","nginx","Redis","MongoDB","Storm","Spark","HBase","Flume","ZooKeeper","Kafka","hive","Bootstrap"," Vue","Apache","nosql","ai","django","AJAX","p2p","Banner","MyBatis","html5","Docker","pr","ps","ae","Unity3D","Cocos"]

headers = {
    'Accept':'application/json, text/plain, */*',
    'User-Agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.131 Safari/537.36',
    'Referer':"https://sou.zhaopin.com/?p=2&jl=489&sf=0&st=0&kw=git&kt=3",
}

# 定义id,便于统计
bb_id = 0
# with open("sao.json","a+",encoding='utf-8',newline='') as fp:
for i in work_page:
    # 请求头
    headers = {
        'Accept': 'application/json, text/plain, */*',
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.131 Safari/537.36',
        'Referer': "https://sou.zhaopin.com/?p=2&jl=489&sf=0&st=0&kw="+i+"&kt=3",
    }
    # 爬取每个职业1000页数据
    for j in range(1,1001):
        # 防止被封，每页休眠0.5秒
        time.sleep(1)
        # start 爬取数量 , PageSize每页数据至多90/页 work_page:爬取职业
        # 自动切换
        url = "https://fe-api.zhaopin.com/c/i/sou?start=" + str(j) + "&pageSize=90&cityId=489&salary=0,0&workExperience=-1&education=-1&companyType=-1&employmentType=-1&jobWelfareTag=-1&kw=" + i + "&kt=3&=0&_v=0.13106924&x-zp-page-request-id=5b19dcb044f14e268fecd1e7fe4c5607-1558011117067-262878"
        # print(url)
        # 防止数据不存在程序中断
        try:
            response_zhaopin = zhaopBse.get(headers=headers, url=url).json()
            infor_datas = response_zhaopin['data']['results']
        except:
            print("有点错误")
            continue

        for infor_data in infor_datas:
            # 职位名称
            job_name = infor_data['jobName']
            # 薪水
            salary = infor_data['salary']
            # 福利待遇
            welfare = "".join(infor_data['welfare'])
            # 工作年限
            workingExp = infor_data['workingExp']['name']
            # 学历要求
            edu = infor_data['eduLevel']['name']
            # 全职or兼职
            emplType = infor_data['emplType']
            # 发布时间
            try:
                createDate = "".join(infor_data['updateDate'])
            except:
                createDate = "缺少发布时间"
            # 公司名称
            company_name = infor_data['company']['name']
            # 公司规模
            company_size = infor_data['company']['size']['name']
            # 公司类型
            company_type = infor_data['company']['type']['name']
            # 公司主页
            company_url = infor_data['company']['url']
            # 详情地址
            position_url = infor_data['positionURL']
            bb_id = bb_id + 1
            data_tmp = {
                "bb_id":bb_id,
                "job_name":job_name,
                "salary":salary,
                "welfare":welfare,
                "workage":workingExp,
                "edu":edu,
                "emplType":emplType,
                "createDate":createDate,
                "company_name":company_name,
                "company_size":company_size,
                "company_type":company_type,
                "company_url":company_url,
                "position_url":position_url
            }
            print(job_name)
            # 填充数据
            cursor.execute(sql,(job_name,salary,welfare,workingExp,edu,emplType,createDate,company_name,company_size,company_type,company_url,position_url))
            # 提交sql
            conn.commit()

conn.close()

```
