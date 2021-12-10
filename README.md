# 工作代碼整理
以下列舉一些工作上所用到之代碼

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

###檢查資料有無回傳###
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
```
## 與MySQL互動計算發電時數，並計算接下來所需維護之時間
#### `從MySQL中撈回發電時數，並且根據所需來計算下幾次需要維護之日期，以提供給頁面呈現讓同仁得知`
```markdown
import pymysql
import pandas as pd
from datetime import datetime, timedelta
import telegram
import requests
import json
import logging
from airflow import DAG
from airflow.operators.python_operator import PythonOperator


default_args = {
    'owner': 'Bo Han Chen',
    'start_date': datetime(2021, 7, 5, 0, 0),
    'retries': 4,
    'retry_delay': timedelta(minutes=1)
}

dag = DAG(
    dag_id='next_run_datetime',
    description='next_run_datetime',
    default_args=default_args,
    schedule_interval='0 8 * * *'
)

###計算發電時數，並自動產生下次維護日期###
def next_run_datetime(**context):
    gas_engine = {
        '139':'field_346_gas_engine_run_hour',
        '140':'field_379_gas_engine_run_hour',
        '141':'field_349_gas_eng1_run_hour',
        '142':'field_354_gas_eng1_run_hour',
        '143':'field_354_gas_eng2_run_hour',
        '144':'field_385_gas_eng1_run_hour',
        '145':'field_385_gas_eng2_run_hour',
        '146':'field_383_gas_eng1_run_hour'
    }
    db_settings = {
        "host": "xx.xx.xxx.xxxx",
        "port": 3306,
        "user": "root",
        "password": "xxxxxxxxxxxxxxxxxxx",
        "db": "greenshepherd",
        "charset": "utf8"
    }
    
    for i in gas_engine:
        ###從MYSQL抓 last_run_hour
        conn = pymysql.connect(**db_settings)
            # 建立Cursor物件
        with conn.cursor() as cursor:
            # 查詢資料SQL語法
            command = "SELECT * FROM greenshepherd.CronItem where id ="+str(i)
            # 執行指令
            cursor.execute(command)
            # 取得所有資料
            result = cursor.fetchall()
            print(result)
        last_run_hour = result[0][7]

        ###從TD抓 剩餘資料###
        url = 'http://xx.xx.xx.xx:xxxx/rest/sql'
        headers={'Authorization': 'Basic cm9vdDp0YW9zZGF0YQ==','Content-Type': 'text/plain'}
        sql = 'select last(ts) from '+gas_engine[i]+' where value <'+str(last_run_hour)+' and value>0'
        try:
            response = requests.get(url,headers=headers, data=sql)
        except Exception as e:
            logging.error(e)
        r = json.loads(response.text, strict=False) #讀取json檔
        last_run_datetime = r['data'][0][0] #上次日期
        last_run_datetime = datetime.strptime(last_run_datetime, "%Y-%m-%d %H:%M:%S.%f")
        sql = 'select last(*) from '+gas_engine[i]+ ' where value >0'
        try:
            response = requests.get(url,headers=headers, data=sql)
        except Exception as e:
            logging.error(e)
        r = json.loads(response.text, strict=False) #讀取json檔
        ###
        now_run_datetime = r['data'][0][0] #現在日期
        now_run_datetime = datetime.strptime(now_run_datetime, "%Y-%m-%d %H:%M:%S.%f")
        
        now_run_hour = r['data'][0][1] #現在時數
        
        hour_date = now_run_datetime-last_run_datetime
        hour_date = hour_date.seconds/3600+hour_date.days*24 #所耗時數
        
        hour_engine = now_run_hour - last_run_hour #發電時數
        
        if hour_engine !=0: 
            date_to_engine=hour_date/hour_engine #每發電時數所耗時數
            
            need_hour = 200-hour_engine #還需發電時數
            
            next_run_datetime = (now_run_datetime+timedelta(hours=need_hour*date_to_engine)).strftime("%Y-%m-%d %H:%M:%S")
            
            print(last_run_hour,now_run_hour,next_run_datetime,now_run_datetime,'-',last_run_datetime)
            if str(datetime.strptime(next_run_datetime, "%Y-%m-%d %H:%M:%S").date()) <= str(datetime.now().date()):
                last_run_hour = now_run_hour

            full_200_hour = 200*date_to_engine
            next_run_datetime = datetime.strptime(next_run_datetime, "%Y-%m-%d %H:%M:%S")
            
            next_item=[str(next_run_datetime.replace(microsecond=0).isoformat())+'Z']
            a = next_run_datetime
            for j in range(0,10):
                a += timedelta(hours=full_200_hour)
                next_item.append(str(a.replace(microsecond=0).isoformat())+'Z')
            last_run_datetime = str(last_run_datetime)

            ###存回資料庫###
            with conn.cursor() as cursor:
                command = f'''UPDATE greenshepherd.CronItem SET last_run_datetime=\"{last_run_datetime}\" ,next_run_datetime=\"{next_run_datetime}\",last_run_hour={last_run_hour},next_items=\"{next_item}\" where id={i}'''
                # 執行指令
                cursor.execute(command)
                # 提交
                conn.commit()
        else:
            continue
        
next_run_datetime = PythonOperator(
    task_id='next_run_datetime',
    python_callable=next_run_datetime,
    provide_context=True,
    dag=dag
)

next_run_datetime

```
## 利用FTP來實現定時上傳資料
#### `此代碼寫入監控軟體中，使用FTP來自動上傳已整理之資料到客戶電腦，並根據要求給予制式化檔名`
```markdown
import os
from ftplib import FTP  #載入ftp模組
from datetime import datetime
import requests
import shutil

###主程序###
@staticmethod
def ftp_data(data, field_id):
    msg_title = "pvdatipp_id,epc_ivt_id,dc_current,dc_kwh,dc_volt,dc_watt,kw"
    msg_title_list = msg_title.split(',')
    msg_data = data
    data_time = datetime.strptime(msg_data['datetime'], "%Y-%m-%d %H:%M:%S")
    ymd = data_time.strftime("%Y-%m-%d")
    hms = data_time.strftime("%H:%M:%S")
    field_id = "field_" + str(field_id)
    all_msg_data = ""
    for i in range(1, 5):
        inverter = "inv_h5_2_" + str(i)
        all_msg_data = all_msg_data + ymd + '\n' + hms + ',' + field_id + ',' + inverter + ','
        for i in range(2, 7):
            all_msg_data += str(msg_data[inverter + ':' + msg_title_list[i]]) + ','
        all_msg_data = all_msg_data + '\n'
            # 檔名
    ip = requests.get('https://api.ipify.org').text
    time = data_time.strftime("%Y%m%d") + str(data_time.hour).zfill(2) + str((data_time.minute // 5) * 5).zfill(
        2) + '00'
    name = field_id + "_" + ip + "_" + time

    text_create(name, msg_title, all_msg_data, data_time)
    # IF 時間 整除30 上傳 所有檔案
    if data_time.minute == 29 or data_time.minute == 59:
        ftp_upload("xxx.xxx.xxx.xx", "john20118", "xxxxxxx", "C:\\Users\\Administrator\\Desktop\\FTP")
        return 'FTP UPLOAD'
    else:
        return 'CREAT TEXT'

###產生檔案代碼###
@staticmethod
def text_create(name, msg_title, all_msg_data, data_time):
    path = "C:\\Users\\gs-pc-user\\Desktop\\FTP\\"
    ftp_dir = "C:\\Users\\gs-pc-user\\Desktop\\FTP"
    if os.path.isdir(ftp_dir) == False:
        os.mkdir(ftp_dir)    
    full_path = path + name + '.txt'
    file = open(full_path, 'a')  # 開啟 寫入模式 無論是否有TXT檔 沒有就會創建議一個
    with open(full_path, 'r') as file:
        contents = file.read()
        file.close()
    if "pvdatipp_id" not in contents:
        file = open(full_path, 'a')
        file.write(msg_title)
        file.write('\n')
        file.write(str(all_msg_data))
        file.write('\n')
        file.close()
    else:
        file = open(full_path, 'a')
        file.write(str(all_msg_data))
        file.write('\n')
        file.close()
    if data_time.minute % 5 == 4:
        file = open(full_path, 'a')
        file.write('#END')
        file.close()

###上傳代碼###
@staticmethod
def ftp_upload(target_ip, target_account, target_password, local_file_path):
    ftp = FTP()  # 設定變數
    ftp.set_debuglevel(2)  # 開啟除錯級別2，顯示詳細資訊
    ftp.connect(target_ip, 21)  # 連線的ftp sever和埠
    ftp.login(target_account, target_password)  # 連線的使用者名稱，密碼
    bufsize = 1024
    # 設定緩衝塊大小
    all_txt = os.listdir(local_file_path)
    for j in all_txt:
        file_name = local_file_path + "\\" + j
        file_handler = open(file_name, 'rb')
        # 以讀模式在本地開啟檔案
        ftp.storbinary('STOR %s' % os.path.basename(file_name), file_handler, bufsize)
        file_handler.close()
    # 上傳所有檔案後 刪除所有檔案
    try:
        shutil.rmtree(local_file_path)
        os.mkdir(local_file_path)
    except OSError as e:
        print(f"Error:{e.strerror}")

```

## 利用[android platform toolsm](https://developer.android.com/studio/releases/platform-tools)與手機互動來實現自動截圖，
#### `使用android platform toolsm來實現截圖，利用python與CMD互動來操作手機並且自動儲存截圖傳回電腦`
```markdown
import os
import time
from datetime import datetime
from PIL import Image

field = ('108B02','107S02','109B02','109B04','109B05')
os.chdir('C:\\Users\\gs-pc-user\\Desktop\\platform-tools') #進入指定目錄
os.popen('adb devices') 
print('開啟APP')
os.popen('adb shell input tap 950 1634') #開啟監控APP
time.sleep(3)
print('回列表頁')
os.popen('adb shell input tap 130 1824') #無論如何都先點一回列表
time.sleep(3)
while True :
    for i in range(0,5):
        high = 180
        datetime.now()
        print(f'進入{field[i]}')
        os.popen(f'adb shell input tap 100 {high+i*140}') #點選案場位置
        t = str(datetime.now().replace(microsecond=0)).replace(':','').replace('-','').replace(' ','')
        time.sleep(30)
        print('--截圖')
        os.popen(f'adb shell /system/bin/screencap -p /sdcard/{t}.png') #
        time.sleep(5)
        print('--存檔')
        os.popen(f'adb pull /sdcard/{t}.png C:/Screen/{field[i]}/{t}.png') #從手機拉圖到電腦
        time.sleep(2)
        print('--開始裁切')
        img = Image.open(f'C:\\Screen\\{field[i]}\\{t}.png') #開啟圖片
        cropped = img.crop((0,450,1100,1600)) #裁切
        cropped.save(f'C:\\Screen\\{field[i]}\\{t}.png') #儲存覆蓋截圖
        print('回列表頁')
        os.popen('adb shell input tap 130 1824') #無論如何都先點一回列表
        time.sleep(3)
    print('等待下一個10分')
    time.sleep(600)
```
# 其他作品
### [地震與不同訊號時間差](https://github.com/john20118/MyPortfolio/blob/master/%E5%9C%B0%E9%9C%87%E8%88%87%E4%B8%8D%E5%90%8C%E5%84%80%E5%99%A8%E4%BF%A1%E8%99%9F.pdf)
#### `從濾波過後的磁場、次聲波、都普勒與地震波來觀察地震發生後不同訊號反應時間差`
### [三分量磁力儀來向](https://github.com/john20118/MyPortfolio/blob/master/%E4%B8%89%E5%88%86%E9%87%8F%E7%A3%81%E5%8A%9B%E5%84%80%E4%BE%86%E5%90%91.pdf)
#### `分析三分量磁力儀X與Y來向，經過濾波後觀察是否能透過X與Y分量來判斷地震方位`
### [棒球恢復係數對ERA(防禦率)之影響](https://github.com/john20118/MyPortfolio/blob/master/AI%E8%B3%87%E6%96%99%E7%A7%91%E5%AD%B8%E4%BA%BA%E6%89%8D%E5%B0%88%E9%A1%8C%E5%A0%B1%E5%91%8A(%E6%A3%92%E7%90%83).pdf)
#### `從各種投手數據來觀察棒球恢復係數是否會影響投手表現`
