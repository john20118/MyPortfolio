# 工作經驗代碼
You can use the [editor on GitHub](https://github.com/john20118/MyPortfolio/edit/gh-pages/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.


## 與TDEngine互動並產生CSV檔

#### `將從TDEngine中撈回的資料整合，並輸出CSV檔提供給客戶`

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
###輸出CSV檔###
df1.to_csv('data.csv')
```

For more details see [Basic writing and formatting syntax](https://docs.github.com/en/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax).
## 與TDEngine互動計算數據，並且利用Airflow自動化
#### `將從TDEngine中撈回的資料計算，並利用Airflow自動排程在將計算過的資料傳回資料庫`
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
## 登入並爬取監控網站資料，傳回資料庫
#### `與需要登入之監控網站互動，爬取其發電資料並且傳回資料庫`
```markdown
import requests
import json
import pandas as pd
from datetime import datetime, timedelta
import logging

###爬從登入得到 ACCESS token###
login_url = 'https://www.xxxxx.com/xxxxxxxxx/xxxxxxxxx/xxxxxxxx'
headers = {
    "Accept-Language": "zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7",
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.183 Safari/537.36'
}
payload={"ClientID": "ro.client",
        "Password": "xxxxxx",
        "Scopes": "openid profile roles read write api offline_access",
        "UserName": "xxxxx"
    }
session = requests.session()
result = session.post(url=login_url, data = payload, headers = headers)
r=json.loads(result.text, strict=False) #讀取json檔
access_token = r['AccessToken']

###進入要爬的頁面###
cookies = {'lang': 'zh'}
headers = {
    'Connection': 'keep-alive',
    'Accept': 'application/json',
    'Authorization': 'Bearer '+access_token,
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.183 Safari/537.36',
    'Content-Type': 'application/json',
    'Origin': 'https://www.mypvms.com',
    'Sec-Fetch-Site': 'same-origin',
    'Sec-Fetch-Mode': 'cors',
    'Sec-Fetch-Dest': 'empty',
    'Referer': 'https://www.mypvms.com/',
    'Accept-Language': 'zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6',
}

data ={'userId': 193, 'methodName': "GetPlantOutPutData", 'queryParames': {'ipp_id': 214}}
response = requests.post('https://xxxxxxx/xxxxxxx/xxxx/xxxxxxxxx', headers=headers, cookies=cookies, data=json.dumps(data))
a=response.json()
a=a['rawData']

###將資料傳回TD資料庫###
for i in range(0,17):
    sql='insert into db.field_274_inv_'+str(i+1)+'_kw USING gs_data TAGS (274,\'107S22\', 1, \'inv\') VALUES (\''+str(datetime.strptime(str(a[i]['pv_date']).replace("/","-"), "%Y-%m-%d %H:%M:%S")-timedelta(hours=8))+'\','+str(a[i]['output_power'])+')'
    url = 'http://xx.xx.xx.xx:xxxx/rest/sql'
    try:
        res = requests.post(
            url,
            headers={
                'Authorization': 'xxxxxxxxxxxxx',
                'Content-Type': 'xxxxxxxxxxxxxxx'
            },
            data=sql
        )
    except Exception as e:
        logging.error(e)

    sql='insert into db.field_274_inv_'+str(i+1)+'_kwh USING gs_data TAGS (274,\'107S22\', 1, \'inv\') VALUES (\''+str(datetime.strptime(str(a[i]['pv_date']).replace("/","-"), "%Y-%m-%d %H:%M:%S")-timedelta(hours=8))+'\','+str(a[i]['ivt_life_power'])+')'
    try:
        res = requests.post(
            url,
            headers={
                'Authorization': 'Basic cm9vdDp0YW9zZGF0YQ==',
                'Content-Type': 'text/plain'
            },
            data=sql
        )
    except Exception as e:
        logging.error(e)
```
## 檢查回傳狀況，若有異常傳訊息給Telegram
#### `檢查所有案場之資料，並因應不同案場狀況來調整代碼。如有資料無回傳則回傳給Telegram`
```markdown
# -*- coding: utf-8 -*-
"""
Created on Tue Mar 16 13:09:09 2021

@author: gs-pc-user
"""

import telegram
from datetime import datetime, timedelta
import requests
import json
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.models import Variable

###Airflow參數###
default_args = {
    'owner': 'Bo Han Chen',
    'start_date': datetime(2021, 3, 17, 2, 0),
    'retries': 4,
    'retry_delay': timedelta(minutes=1)
}
dag = DAG(
    dag_id='message',
    description='field_reply',
    default_args=default_args,
    schedule_interval='0 */1 * * *'
)

def reply(**context):
    ###資料表名稱###
    field={'107S02 合鑫':{'field_346_':['inv_1','inv_2','inv_3','inv_4','inv_5','inv_6','inv_7','inv_8','inv_9','inv_10','inv_1_inv','gas_engine_error_code'],'_kwh':[]},
            '106S10 修平':{'field_7_':['inv_1','inv_2','inv_3','inv_4','inv_5','inv_6','inv_7','inv_8','inv_9','inv_10','inv_11'],'_kwh':[]},
            '107S11 南雄':{'field_200_':['inv_1','inv_2','inv_3','inv_4','inv_5','inv_6','inv_7','inv_8','inv_9','inv_10','inv_11'],'_kwh':[]},
            '107S13 中正':{'field_196_':['inv_1','inv_2'],'_kwh':[]},
            '109S02 太億':{'field_180_':['inv_1','inv_2'],'_kwh':[]},
            #'107S12 三民(暫不處理)':{'field_304_':['inv_1','inv_2','inv_3','inv_4','inv_5','inv_6'],'_dc_kwh':[]},
            '106S08 馬清洲':{'field_272_':['inv_1','inv_2'],'_dc_kwh':[]},
            '105G01 台鉅':{'field_43_':['inv_h5_1','inv_h5_2','inv_h5_3','inv_h5_4','inv_h5_5','inv_h5_6','inv_h5_7','inv_h5_8','inv_h5_9','inv_h5_10','inv_h5_11','inv_h5_12','inv_h5_13'],'_dc_kwh':[]},
            '105S07 六和_中壢':{'field_277_':['inv_h5_1','inv_h5_2','inv_h5_3','inv_h5_4','inv_h5_5','inv_h5_6','inv_h5_7','inv_h5_8','inv_h5_9','inv_h5_10','inv_h5_11','inv_h5_12','inv_h5_13','inv_h5_14','inv_h5_15','inv_h5_16'],'_dc_kwh':[]},
            '105S07-1 六和_新屋':{'field_284_':['inv_h5_1','inv_h5_2','inv_h5_3','inv_h5_4','inv_h5_5','inv_h5_6','inv_h5_7','inv_h5_8','inv_h5_9','inv_h5_10','inv_h5_11','inv_h5_12','inv_h5_13','inv_h5_14','inv_h5_15'],'_dc_kwh':[]},
            '105S16 劉慧楨':{'field_268_':['inv_h5_1','inv_h5_2'],'_dc_kwh':[]},
            'lg2018S07 江東軒':{'field_373_':['inv_1','inv_2','inv_3','inv_4'],'_dc_kwh':[]},
            'lg2018S10 許金發':{'field_290_':['inv_1','inv_2','inv_3','inv_4','inv_5','inv_6','inv_7','inv_8','inv_9','inv_10'],'_dc_kwh':[]},
            '109B02 良輝':{'field_354_':['inv_1','inv_2','gas_eng1_error_code','gas_eng2_error_code'],'_kwh':[]},
            '108B02 章勳':{'field_379_':['inv_1','gas_engine_error_code'],'_kwh':[]},
            '109B05 張重興':{'field_383_':['inv_1'],'_kwh':[]},
            '109B04 張林素梅':{'field_385_':['inv_1','inv_2','gas_eng1_error_code','gas_eng2_error_code'],'_kwh':[]},
            '109B01 承新':{'field_349_':['inv_1'],'_kwh':[]},
            '109H15 宏遠':{'field_82_ch':['09','15'],'_q':[]},
            '106H13 輔仁':{'field_372_':['water'],'_q':[]},
            '110S02 六和_中壢_ST':{'field_277_':['inv_h5_2_1','inv_h5_2_2','inv_h5_2_3','inv_h5_2_4','inv_h5_2_5','inv_h5_2_6','inv_h5_7','inv_h5_8','inv_h5_9','inv_h5_10','inv_h5_11','inv_h5_12','inv_h5_13','inv_h5_14','inv_h5_15','inv_h5_16'],'_dc_kwh':[]},
            '110S02 六和_中壢_SW':{'field_277_':['inv_h5_2sw_1','inv_h5_2sw_2','inv_h5_2sw_3','inv_h5_2sw_4','inv_h5_2sw_5','inv_h5_2sw_6','inv_h5_2sw_7','inv_h5_2sw_8','inv_h5_2sw_9','inv_h5_2sw_10','inv_h5_2sw_11','inv_h5_2sw_12'],'_dc_kwh':[]},
            '110S02 六和_新屋':{'field_284_':['inv_h5_2_1','inv_h5_2_2','inv_h5_2_3','inv_h5_2_4'],'_dc_kwh':[]},
           }
    ###時間段調整###
    d1 = datetime.today().strftime("%Y-%m-%d %H") #當前時間
    d2 = str(datetime.strptime(d1, "%Y-%m-%d %H")-timedelta(hours=8)) #往前8小時
    ccc=[] #空list，用來裝所有訊息
    ###分別檢查每個案場###
    for i in field:
        ii=i.split('.') #利用spilt 把str變成list
        a = []#空list
        bbb = []#空list
        aaa = []#空list
        eee = []
        bb = ''
        for j in field[i]:
            a.append(j) #把field裡的key放到a
        ###拉資料###   
        num = 0
        nnum = 0
        ###檢查當日每小時之8筆資料###
        for k in field[i][a[0]]: #取出key裡面的值
            if k == 'gas_engine_error_code' or k == 'gas_eng1_error_code' or k == 'gas_eng2_error_code':
                sql = 'select * from '+a[0]+k+' where ts < "'+d1+':00:00'+'" and ts > "'+d2+'" order by ts desc'
            else:
                sql = 'select max(value) from '+a[0]+k+a[1]+' where ts < "'+d1+':00:00'+'" and ts > "'+d2+'" interval(60m) order by ts desc limit 8'
            #return sql
            url = 'http://xx.xx.xx.xx:xxxx/rest/sql'
    
            response = requests.get(
                url,
                headers={
                    'Authorization': 'Basic cm9vdDp0YW9zZGF0YQ==',
                    'Content-Type': 'text/plain'
                },
                data=sql
            )
    
            try:
                r=json.loads(response.text, strict=False) #讀取json檔
                df=r['data']
            except:
                df = []
            itsnotok=''
            itsok=''
            ###如果有資料###
            if len(df) > 0:
                num += 1
                if k == 'gas_engine_error_code' or k == 'gas_eng1_error_code' or k == 'gas_eng2_error_code':
                        if df[0][1] > 0:
                            aa='{} {} {}'.format(k,': ',str(df[0][1]))
                            aaa += aa.split('*')  #利用spilt 把str變成list         
            else:
                nnum += 1
                if k == '09':
                    itsnotok='C '
                elif k == '15':
                    itsnotok='A '
                else:
                    itsnotok=k+' '
            if itsnotok != '':
                bb += itsnotok
        if num == len(field[i][a[0]]):
            continue
            #itsok='ok'
            #eee += itsok.split('.')
        ###無回傳資料數量等於感測器數量則直接顯示都沒回傳
        if nnum == len(field[i][a[0]]):
            itsok='所有感測器都沒回傳'
            eee += itsok.split('.')
        ###無回傳資料數量不等於感測器數量則顯示那些無回傳
        else:
            if bb != '':
                bb = bb+':沒有回傳'
                bbb = bb.split('O') #利用spilt 把str變成list
        dd=ii+eee+bbb+aaa #每個案場資訊加總
        ccc+=dd #全部案場加總        ###送給機器人###
        ###取回Airflow內之舊訊息
        field_reply_old = Variable.get("field_reply")
        ###設置新訊息
        field_reply = Variable.set("field_reply_new", ccc)
    return ccc,field_reply, field_reply_old
    
###與Airflow內之舊訊息比較 若有不同則傳送新訊息
def send_message(**context):
    ccc,field_reply,field_reply_old = context['task_instance'].xcom_pull(task_ids='reply')
    field_reply_new = Variable.get("field_reply_new")
    if field_reply_new != field_reply_old:
        Variable.set("field_reply", ccc)
        bot = telegram.Bot(token=('****************************'))
        bot.sendMessage('-**********', "\n".join(ccc))
        #bot.sendMessage('-221602403', field_reply_old)
        

reply = PythonOperator(
    task_id='reply',
    python_callable=reply,
    provide_context=True,
    dag=dag
)

send_message = PythonOperator(
    task_id='send_message',
    python_callable=send_message,
    provide_context=True,
    dag=dag
)

reply >> send_message
