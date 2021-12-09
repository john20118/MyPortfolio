## 工作經驗代碼
You can use the [editor on GitHub](https://github.com/john20118/MyPortfolio/edit/gh-pages/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.


### 與TDEngine互動並產生CSV檔

##### 目的:將從TDEngine中撈回的資料整合，並輸出CSV檔提供給客戶

```markdown
from datetime import datetime, timedelta
import logging
import requests
import pandas as pd
import json
###時間段更改###
start_time = "2021-11-09 16:00:00"
end_time =  "2021-11-20 16:00:00"
####與TDEngine互動###
url = 'http://xx.xx.xx.xx:xxxx/rest/sql'
headers={'Authorization': '*****','Content-Type': '*****'}
columns = ['gas_eng1_kw_b','gas_eng1_kw_c','gas_eng1_hz','gas_eng2_kw_a','gas_eng2_kw_b','gas_eng2_kw_c','gas_eng2_hz','inv_2_kw']
###TD語法拉回資料，並且利用PD將不同欄位資料合併
sql = 'select * from field_385_gas_eng1_kw_a where ts > "' +start_time+ '" and ts < "' + end_time + '"'
try:
    response = requests.get(url,headers=headers, data=sql)
except Exception as e:
    logging.error(e)
r = json.loads(response.text, strict=False) #讀取json檔
data = r['data']
df1 = pd.DataFrame(data)
df1 = df1.set_index(df1[0]).drop(columns=[0]).rename(columns={1:'gas_eng1_kw_a'})
for i in columns:
    sql = 'select * from field_385_'+i+' where ts > "' +start_time+ '" and ts < "' + end_time + '"'
    try:
        response = requests.get(url,headers=headers, data=sql)
    except Exception as e:
        logging.error(e)
    r = json.loads(response.text, strict=False)
    data = r['data']
    df = pd.DataFrame(data)
    df = df.set_index(df[0]).drop(columns=[0]).rename(columns={1:i})
    df1 = df1.merge(df, how='left', left_index=True, right_index=True)
df1 = df1.reset_index()
###將UTC+0轉回UTC+8
df1[0] = pd.to_datetime(df1[0]) + timedelta(hours=8)
df1 = df1.set_index(df1[0]).drop(columns=[0])
###輸出CSV黨###
df1.to_csv('data.csv')
```

For more details see [Basic writing and formatting syntax](https://docs.github.com/en/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax).
### 與TDEngine互動計算數據，並且利用Airflow自動化
##### 目的:將從TDEngine中撈回的資料計算，並利用Airflow自動排程在將計算過的資料傳回資料庫
```markdown
from datetime import datetime, timedelta
import logging
import requests
import json
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
###Airflow參數###
default_args = {
    'owner': 'Bo Han Chen',
    'start_date': datetime(2021, 10, 28, 0, 0),
    'retries': 4,
    'retry_delay': timedelta(minutes=1)
}
dag = DAG(
    dag_id='110S02',
    description='my dag',
    default_args=default_args,
    schedule_interval='*/15 * * * *'
)
url = 'http://xx.xx.xx.xx:xxxx/rest/sql'
headers={'Authorization': '******','Content-Type': '****'}
"""
六和二期總發電量
"""
def total_kwh(**context):
    d1 =datetime.today().strftime("%Y-%m-%d %H:%M:%S")   
    field = ['field_277_inv_h5_2sw','field_277_inv_h5_2','field_284_inv_h5_2']
    field_into = ['field_277_inv_total_kwh_sw','field_277_inv_total_kwh_st','field_284_inv_total_kwh_2']
    inv_num = [12,6,4]
    for i in range(0,3):
        total_kwh = 0
        for j in range(1,inv_num[i]+1):
            sql = f'select last(value) from {field[i]}_{str(j)}_dc_kwh' 
            try:
                response = requests.get(url,headers=headers, data=sql)
            except Exception as e:
                logging.error(e)
            r = json.loads(response.text, strict=False) #讀取json檔
            kwh = r['data'][0][0] #拉出DATA
            total_kwh += kwh
        sql=f'insert into db.{field_into[i]} VALUES(\'{str(d1)}\',{total_kwh})'
        try:
            response = requests.get(url,headers=headers, data=sql)
        except Exception as e:
            logging.error(e)     
"""
六和二期每日發電量
"""
def today_kwh(**context):
    field = ['field_277_inv_h5_2sw','field_277_inv_h5_2','field_284_inv_h5_2']
    field_into = ['field_277_inv_today_kwh_sw','field_277_inv_today_kwh_st','field_284_inv_today_kwh_2']
    inv_num = [12,6,4]
    d1 =datetime.today().strftime("%Y-%m-%d %H:%M:%S")       
    for i in range(0,3):
        today_kwh = 0
        for j in range(1,inv_num[i]+1):
            if datetime.today().hour >= 16:
                d2 =datetime.today().strftime("%Y-%m-%d") #目前時間 並且轉換格式
                sql = f'select last(value) from {field[i]}_{str(j)}_today_kw where ts > \'{d2} 16:00:00\''
            else:
                d2 = (datetime.today() - timedelta(days=1)).strftime("%Y-%m-%d")
                sql = f'select last(value) from {field[i]}_{str(j)}_today_kw where ts > \'{d2} 16:00:00\'' 
            try:
                response = requests.get(url,headers=headers, data=sql)
            except Exception as e:
                logging.error(e)
            try:
                r = json.loads(response.text, strict=False) #讀取json檔
                kwh = r['data'][0][0] #拉出DATA
                today_kwh += kwh
                print(today_kwh)
            except:
                pass
        sql=f'insert into db.{field_into[i]} VALUES(\'{str(d1)}\',{today_kwh})'
        try:
            response = requests.get(url,headers=headers, data=sql)
        except Exception as e:
            logging.error(e) 
"""
六和二期總WATT
"""
def today_watt(**context):
    d1 =datetime.today().strftime("%Y-%m-%d %H:%M:%S")   
    field = ['field_277_inv_h5_2sw','field_277_inv_h5_2','field_284_inv_h5_2']
    field_into = ['field_277_inv_total_watt_sw','field_277_inv_total_watt_st','field_284_inv_total_watt_2']
    inv_num = [12,6,4]  
    for i in range(0,3):
        total_watt = 0
        for j in range(1,inv_num[i]+1):
            sql = f'select last(value) from {field[i]}_{str(j)}_dc_watt' 
            try:
                response = requests.get(url,headers=headers, data=sql)
            except Exception as e:
                logging.error(e)
            r = json.loads(response.text, strict=False) #讀取json檔
            watt = r['data'][0][0] #拉出DATA
            total_watt += watt
        sql=f'insert into db.{field_into[i]} VALUES(\'{str(d1)}\',{total_watt})'
        try:
            response = requests.get(url,headers=headers, data=sql)
        except Exception as e:
            logging.error(e)
###以何種形式跑代碼###
total_kwh = PythonOperator(
    task_id='total_kwh',
    python_callable=total_kwh,
    provide_context=True,
    dag=dag
)           
today_kwh = PythonOperator(
    task_id='today_kwh',
    python_callable=today_kwh,
    provide_context=True,
    dag=dag
)
today_watt = PythonOperator(
    task_id='today_watt',
    python_callable=today_watt,
    provide_context=True,
    dag=dag
)
###代碼順序###
total_kwh > today_kwh > today_watt
```
### Support or Contact
Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
