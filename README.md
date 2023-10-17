# 爬蟲抓取ptt熱門文章
# 解決問題
1. 以往要找取熱門的文章，只能一頁一頁找，不方便且耗時，使用這個軟體，可以快速掌握當天討論度最高的文章，節省非常多的時間。

# 作品介紹

1. 執行主程式使用爬蟲爬取ppt熱門文章後，會產生一個csv檔案。
2. 此csv檔案，會抓取推文數最高的前N名文章(代表最熱門的前N名文章)，並且同時抓取推文數，標題，文章連結，作者，內文，和發佈時間。
4. 我們的搜尋範圍為頁數，可在setting.py 檔案中做調整，也可修改篩選前100名或是前10名的文章。
5. 使用者可以點選文章連結直接連到網站。

![](https://i.imgur.com/opnMNqt.png)



# 操作說明🛠

主要檔案為兩個
1. main.py 為主程式，我們需要去執行他，下載後在同個資料夾路徑下，開起終端機，然後輸入
`python main.py` 執行後即可進行爬蟲
2. setting.py 為我們參數值的設定用的檔案。
3. PTT\_tool\_env.yml 這個檔案是這個程式的執行環境，大家可以下載後放到anaconda，讓後再去執行他。至於要怎麼放，大家可以參考這篇文章，或者是直接手動import 因為相關的特別套件也不多，所以因該不會太麻煩。
https://medium.com/qiubingcheng/%E5%A6%82%E4%BD%95%E5%AE%89%E8%A3%9Danaconda-%E4%B8%A6%E4%B8%94%E5%8C%AF%E5%85%A5conda%E8%99%9B%E6%93%AC%E7%92%B0%E5%A2%83-ba2e140706a3

![](https://i.imgur.com/qpXYWpw.png)


# setting.py參數檔案設定
打開檔案後，小弟有把相關參數的註解下下來 ，
相關參數功能，可以參考註解上的內容，然後這個setting.py檔案會被main.py 主程式引入。
```python=
url="https://www.ptt.cc/bbs/studyabroad/index.html"  # 你要爬取的網站的網址，預設為ptt 留學版


page=5 # 為你要往後爬取的頁數

sort_number=10 # 為你要取出的push數最多的前幾篇文章，預設為前10名

file_name="result.csv" #為產生的csv檔案路徑以及名稱，預設為在同個資料夾

```



# 程式碼說明 

### payload 部分
1. 這邊使用payload 去向server傳送請求 並且使用headers 讓我們在抓取資料的時候不要被系統發現是用程式去抓的

2. 可以看到表單是以POST的形式傳送，確認預設的值是'yes'，
所以接下來我們要帶著建立的session，以POST的方式帶著參數登入，再用cookie以GET的方式帶著參數進入主頁。

``` python=
payload = {"from": "https://www.ptt.cc/bbs/Gossiping/index.html","yes": "yes"} 
my_headers = {"user-agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.61 Safari/537.36"}

rs = requests.Session() 
rs.post("https://www.ptt.cc/", data = payload, headers = my_headers) 
```

### 蒐集前n頁文章方法&往前一頁爬蟲方法 
1. 首先，這邊先去使用bs4.BeautifulSoup解析了url這個網址(這個網址是第一頁)。 
2. 那再來會去把這個網頁標籤title下的連結(文章連結)加入link這個list。 
3. 然後再去使用soup.select()把主畫面中的上一頁按鈕的連結抓下來，回傳回去，這樣下一次做for loop 就可以把這個連結當作新的url ，這樣就完成了往前一頁爬蟲的動作。 

```python=
def get_date(url): 
    response = rs.get(url, headers=my_headers)
    soup = bs4.BeautifulSoup(response.text, "html.parser")           #透過BeautifulSoup並用"html.parser"去解析此網站
    titles = soup.find_all("div", class_ = "title")          #抓取每篇文章連結
    #print(titles)
    dates = soup.find_all("div", class_="date")       #抓取日期
    #print(dates)
    print(" 爬蟲進行中....")
    for i in range(len(titles)):                         #用title當文章數量的計數器 去一一檢視日期 
            if(titles[i].a):                           #使用.a是避免已經被刪除的文章
                #print(i)
                # 把所有文章的連結都加入link 
                link.append("https://www.ptt.cc/" + titles[i].find("a").get("href")) # 得到a標籤下的href  
                
    nextlink_2=soup.select('div.btn-group > a')           # 由籤div.btn.group 到標籤a 
    up_page_href = nextlink_2[3]['href'] #在標籤div.btn.group下[0]是看板連結  [1]是精華區連結 [2]是最舊 [3]是上頁 [4]是下頁 [5]是最新
    return  up_page_href  #回傳上一頁的網址 讓下面的for loop 遞迴成功
```

###  了解原始網頁中的html結構 

這邊我們需要去觀察原始網頁中的html結構，方便我們了解要抓取html標籤的哪個部分。

在標籤div.btn.group下
[0]['href'] 是看板連結 
[1]['href'] 是精華區連結
[2]['href'] 是最舊 
[3]['href'] 是上頁 
[4]['href'] 是下頁
[5]['href'] 是最新

![](https://hackmd.io/_uploads/rkOTvWh-p.png)


### (資料處理)依據留言數量的順序來排序抓取到的文章

1.我們這邊設置了二維list，score[j]是第j個文章，score[j][0]存放蓋樓留言數量。score[j][1]存放此文章連結
2. 因為留言數量存放在個別文章的html中，所以必須再針對每個文章的連結做一次解析，然後抓取留言數量的標籤，存放進入score[j][0]
3. 最後使用sort()，針對留言數量，把這個二維list做排序，這樣就會得到依據留言數量大小來排序的所有抓取文章。

```python=
push = [] #設定一個push的串列 去儲存蓋樓數
#score[j]是第j個文章， score[j][0] 和score[j][1] 是存放蓋樓數量 還有此網頁的link

score = [[0 for j in range(2)] for k in range(300)] #設定一個雙重陣列去儲存排序的網址和蓋樓數 方便去比對
for j in range(len(link)):  #把迴圈的range設定成我們擁有的網址的數量  
    response = rs.get(str(link[j]), headers = my_headers) # 針對link中每一篇文章，從心再去訪問一次
    soup = bs4.BeautifulSoup(response.text, "html.parser") #得到文章內頁的原始碼存到soup
    push = soup.find_all("div", class_ = "push") #把蓋樓的數量存到變數push中 所有push的標籤都會被抓出來，所以可以用len(push)看總數量
    
    score[j][0] = len(push) #取出蓋樓數量 並存入score list的0位置
    score[j][1] = link[j] #把link存入score list的1位置
    score.sort(key=lambda x:x[0], reverse =True) #使用python的串列排序的函式sort ，根據蓋樓數量排序好   

```


### 寫入csv檔案部分 
1. 因為我們前面所儲存的是連結，所以這裡要再使用 bs4.BeautifulSoup 解析網頁原始碼
2. 這邊會做一個for loop 代表使用者想要前n篇文章。
3. 並且把文字多餘的部分做slice切割，最後存入我們的csv檔案
```python=
with open(setting.file_name, "a", newline="", encoding='utf-8-sig')as csvfile:
    writer = csv.writer(csvfile)
    writer.writerow(['爬蟲爬取時間',"推文數", "標題","文章連結","作者:", "內文","發布時間"])
    sort_number=10
    for n in range(sort_number): #用迴圈印出抓取之前10個推需文數最多的文章
        response = rs.get(score[n][1], headers = my_headers)  # n是文章數量，[n][1]是網址連結，[n][0] 是push數量。
        soup = bs4.BeautifulSoup (response.text, "html.parser") #再對網站做一次解析
        header = soup.find_all("span", "article-meta-value") #這邊get value 的定義是指把所有的article-meta-value標籤都抓下來存成list 

        author = header[0].text #存入作者  這邊分別使用header[]是因為他分別存在article-meta-value下 
        title = header[2].text #存入標題
        date = header[3].text #存入日期

        main_container = soup.find(id = "main-container",) #抓出內文
        #print(main_container)
        content = main_container.text.replace("\u6ca1", "  ") 
        #print(content)
   
        all_content = content
        pre_content = all_content.split("--")[0]
        texts = pre_content.split('\n')
        contents = texts[2:]
        final_contents = "\n".join(contents)

        writer.writerow([time_today,score[n][0], title,score[n][1],author, final_contents,date]) #寫入csv檔
        print(score[n][1])
print("恭喜完成文章爬蟲,請去result.csv觀看結果")
```



以上為此程式碼的介紹！