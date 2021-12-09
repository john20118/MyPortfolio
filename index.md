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

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/john20118/MyPortfolio/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
