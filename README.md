# selenium--
#用selenium爬取拉钩网有关python的招聘信息
import re
import time
import pymongo
from config import *
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

browser = webdriver.Chrome()
wait = WebDriverWait(browser,10)

client  =pymongo.MongoClient(MONGO_URL)
db = client[MONGO_DB]


def login():
    print('正在登陆....')
    print('正在下载————————————')
    try:
        browser.get('https://www.lagou.com/')
        extra = wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR,'#changeCityBox > p.checkTips > a')))
        extra.click()
        sub = wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR,'#lg_tbar > div > ul > li:nth-child(1) > a')))
        sub.click()
        user = wait.until(EC.presence_of_element_located((By.CSS_SELECTOR,'body > section > div.left_area.fl > div:nth-child(2) > form > div:nth-child(1) > input')))
        user.click()
        user.send_keys(用户名)
        time.sleep(2)
        passwd = wait.until(EC.presence_of_element_located((By.CSS_SELECTOR,'body > section > div.left_area.fl > div:nth-child(2) > form > div:nth-child(2) > input')))
        passwd.click()
        passwd.send_keys(密码)
        submit = wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR,'body > section > div.left_area.fl > div:nth-child(2) > form > div.input_item.btn_group.clearfix > input')))
        submit.click()
        search = wait.until(EC.presence_of_element_located((By.CSS_SELECTOR,'#search_input')))
        button = wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR,'#search_button')))
        search.send_keys('python')
        button.click()
        get_position()
    except TimeoutError:
        login()


def next_pagr():
    print('正在翻页')
    try:
        next_page = wait.until(EC.presence_of_element_located((By.CSS_SELECTOR,'#s_position_list > div.item_con_pager > div > span.pager_next')))
        next_page.click()
        get_position()
    except TimeoutError:
        next_pagr()


def get_position():
    wait.until(EC.presence_of_element_located((By.CSS_SELECTOR,'.s_position_list  .item_con_list li')))
    html = browser.page_source
    # print(html)
    pattern = re.compile('<li.*?180px;">(.*?)</h3>.*?"add">(.*?)</span>.*?class="money">(.*?)</span>.*?</i>(.*?)</div>.*?class="company">.*?0">(.*?)</a>.*?class="industry">(.*?)</div>.*?li_b_r">“(.*?)”</div>.*?</li>',re.S)
    result = re.findall(pattern,html)
    for item in result:
        # print(item)
        position = {
            'work':item[0],
            'place':item[1].replace('[<em>','').replace('</em>]',''),
            'salary':item[2],
            'work_required':item[3].replace('-->','').replace('\n                    ',''),
            'company':item[4],
            'company_type':item[5].replace('\n                    ','').replace('\n                ',''),
            'bounces':item[6]
            }
        # print(position)
        save_to_mongo(position)

def save_to_mongo(result):
    try:
        if db[MONGO_TABLE].insert(result):
            print('存储到MONGO数据库成功',result)
    except Exception:
        print('存储失败',result)


def main():
    login()
    while True:
        next_pagr()
        time.sleep(5)


if __name__ == '__main__':
    main()
