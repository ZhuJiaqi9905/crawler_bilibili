# %%
from selenium.webdriver.support.wait import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.common.by import By
from selenium import webdriver
import requests
import os
import re
import json
import pymysql
import random
import sys
import time
import jieba
import wordcloud
import matplotlib.pyplot as plt

# %%


class MysqlProxy():
    def __init__(self, host, port, user, password, db):
        self.host = host
        self.port = port
        self.user = user
        self.password = password
        self.db = db

    def connect(self):
        try:
            self.conn = pymysql.connect(
                host=self.host, port=self.port, user=self.user, passwd=self.password, db=self.db)
            self.cursor = self.conn.cursor()
        except Exception as e:
            self.conn = None
            self.cursor = None
            print('error in connect mysql', e)

    def createTable(self, table_sql):
        try:
            self.cursor.execute(
                'CREATE TABLE IF NOT EXISTS {}'.format(table_sql))
            self.conn.commit()
        except Exception as e:
            print("error in create table", table_sql, e)

    def insert(self, table, data):
        try:
            keys = ','.join(data.keys())
            values = ','.join(['%s']*len(data))
            sql = 'INSERT INTO {table} ({keys}) VALUES({values})'.format(
                table=table, keys=keys, values=values)
            self.cursor.execute(sql, tuple(data.values()))
            self.conn.commit()
        except Exception as e:
            print(data)
            print('error in insert', sql, e)
            self.cursor.rollback()

    def select(self, sql):
        try:
            self.cursor.execute(sql)
            row = self.cursor.fetchall()
            return row
        except Exception as e:
            print('error in select', sql, e)

# %%

# 创建数据库表


def create_database():
    db = pymysql.connect(host="localhost", user="root",
                         password="TaKagi9905", port=3306)
    cursor = db.cursor()  # 开启游标功能，创建游标对象
    cursor.execute("SELECT VERSION()")
    data = cursor.fetchone()
    print("Database version: ", data)
    cursor.execute("CREATE DATABASE bilibili DEFAULT CHARACTER SET utf8mb4")
    db.close()
# %%

# 爬虫


class Crawler():
    def __init__(self, config, mysql):
        self.base_url = config["base_url"]
        self.header = config["header"]
        self.param = config["param"]
        self.mysql = mysql
        self.mysql.connect()

    def get_text(self, param):
        r = requests.get(self.base_url, params=param, headers=self.header)
        return r.text

    def get_rank_pages(self, table, start=1, end=167):
        for page in range(start, end + 1):
            # 对爬取的每一页解析内容
            self.param["page"] = str(page)
            web_text = self.get_text(self.param)
            js = json.loads(web_text)
            for ele in js["data"]["list"]:
                item = {}
                item["title"] = ele["title"]
                item["badge"] = ele["badge"]
                result = re.findall(r"\d+", ele["index_show"])
                if len(result) > 0:
                    item["index_num"] = int(result[0])
                else:
                    item["index_num"] = None
                item["finish"] = ele["is_finish"]
                result = re.findall(r"\d+", ele["order"])
                if len(result) > 0:
                    item["order_num"] = float(result[0])
                    if not "万" in ele["order"]:
                        item["order_num"] /= 10000
                else:
                    item["order_num"] = None
                item["media_id"] = ele["media_id"]
                for key in item.keys():
                    if item[key] == '':
                        item[key] = None
                self.mysql.insert(table, item)
            print("page ", page)
            time.sleep(random.randint(3, 7))


# %%
# 建立数据库表
mp = MysqlProxy(host="localhost", port=3306, user="root",
                password="TaKagi9905", db="bilibili")
mp.connect()
mp.createTable('''animate (id INT UNSIGNED AUTO_INCREMENT,
                            title VARCHAR(50) NOT NULL,
                            badge VARCHAR(20),
                            index_num INT,
                            finish TINYINT,
                            order_num FLOAT,
                            media_id INT,
                            PRIMARY KEY(id) )''')

# %%
table = 'animate'
config = {
    "header": {
        'user-agent': 'Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36'
    },
    "base_url": "https://api.bilibili.com/pgc/season/index/result",
    "param": {
        "season_version": "-1",
        "area": "-1",
        "is_finish": "-1",
        "copyright": "-1",
        "season_status": "-1",
        "season_month": "-1",
        "year": "-1",
        "style_id": "-1",
        "order": "3",
        "st": "1",
        "sort": "0",
        "season_type": "1",
        "pagesize": "20",
        "page": "1",
        "type": "1"
    }
}
msql = MysqlProxy(host="localhost", port=3306, user="root",
                  password="TaKagi9905", db="bilibili")
c = Crawler(config, msql)
c.get_rank_pages(table=table, start=2, end=160)  # 爬取排行榜信息


# %%

# 统计追番人数超过100万的动漫中，不同播放类型（会员专享/独家等）所占的比例。
mp = MysqlProxy(host="localhost", port=3306, user="root",
                password="TaKagi9905", db="bilibili")
mp.connect()
row = mp.select(
    'SELECT badge, COUNT(badge) FROM animate WHERE order_num > 100 GROUP BY badge')
data = []
label = []
for kv in row:
    if kv[0] != None:
        label.append(kv[0])
        data.append(kv[1])
# 可视化：圈图
plt.rcParams['font.sans-serif'] = 'simhei'
plt.rcParams['axes.unicode_minus'] = False

plt.pie(data, pctdistance=0.8, autopct='%.1f%%')
plt.pie([1], radius=0.6, colors='w')
plt.legend(label, loc='upper right')
plt.savefig('figure/circle.png', dpi=600)
plt.show()

# %%
# 用selenium爬取动漫评论

media_id = mp.select('SELECT media_id FROM animate WHERE title="鬼灭之刃"')[0][0]
file = str(media_id) + '.txt'
url_review = 'https://www.bilibili.com/bangumi/media/md' + media_id + '/#short'
with open(file=file, mode='w', encoding='utf-8') as f:  # 把评论存储到文件中
    browser = webdriver.Chrome()
    try:
        browser.get(url_review)  # 打开网页
        for idx in range(0, 60):
            # 自动下拉滚动条
            js = "var q=document.scrollingElement.scrollTop=10000000"
            browser.execute_script(js)
            time.sleep(random.randint(3, 7))
        # 定位到评论的内容
        elements = browser.find_elements_by_xpath(
            '//*[@class="media-tab-module-content"]/div/div/ul/li/*[@class="review-detail"]/div')
        for element in elements:  # 存储评论
            f.write(element.text)
            f.write('\n')
    finally:
        browser.close()

# %%
# 读取评论，制作词云
with open(file, mode='r', encoding='utf-8') as f:
    lines = f.read()
    words = jieba.lcut(lines)
    reviews = " ".join(words)
# 词云图设置
wc = wordcloud.WordCloud(width=800,
                         height=600,
                         background_color='white',
                         font_path='msyh.ttc',
                         scale=15,
                         stopwords=set([line.strip() for line in open('stop_words.txt', mode='r', encoding='utf-8').readlines()]))
# 给词云图输入文字
wc.generate(reviews)
# 保存词云图
wc.to_file('figure/output.png')
