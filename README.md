# 爬虫（Driver+bs>>>bilibili）
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException
import xlwt
import bs4
from selenium.webdriver.chrome.options import Options

# 浏览器初始化
options = Options()
options.add_argument('--headless')
browser=webdriver.Chrome(options=options)
WAIT=WebDriverWait(browser,10)
browser.set_window_size(1400, 900)
browser.get('https://www.bilibili.com/')

# 数据库初始化

def searchB(src):

    # 表格初始化(仅用于表格储存信息模式)
    book = xlwt.Workbook(src, style_compression=0)
    sheet = book.add_sheet(src, cell_overwrite_ok=True)
    sheet.write(0, 0, '名称')
    sheet.write(0, 1, '地址')
    sheet.write(0, 2, '描述')
    sheet.write(0, 3, '观看次数')
    sheet.write(0, 4, '弹幕数')
    sheet.write(0, 5, '发布时间')
    sheet.write(0, 6, 'Up主')
    n = 1


    def search():
        try:
            print("开始尝试访问b站...")
            input = WAIT.until(EC.presence_of_element_located((By.CSS_SELECTOR, "#nav_searchform > input")))
            submit = WAIT.until(EC.element_to_be_clickable((By.CSS_SELECTOR, '#nav_searchform > div > button')))

            input.send_keys(src)
            submit.click()

            print("搜索成功，转到新窗口")
            all_h = browser.window_handles
            browser.switch_to.window(all_h[1])
            getPage()

            total = WAIT.until(EC.presence_of_element_located((By.CSS_SELECTOR,
                                                               '#all-list > div.flow-loader > div.page-wrap > div > ul > li.page-item.last > button'))).text
            print('总页数为' + total)
            return int(total)
        except TimeoutException:
            print('访问超时，尝试重新访问...')
            return search()

    def getPage():
        WAIT.until(EC.presence_of_element_located((By.CSS_SELECTOR, '#all-list > div.flow-loader > div.filter-wrap')))
        html = browser.page_source
        soup = bs4.BeautifulSoup(html, 'lxml')
        save_data_to_excel(soup)

    def save_data_to_excel(soup):
        list = soup.find(class_='video-list clearfix').find_all(class_='video-item matrix')
        for item in list:
            item_title = item.find('a').get('title')
            item_link = item.find('a').get('href')
            item_des = item.find(class_='des hide').text.strip()
            item_playtime = item.find(class_='so-icon watch-num').text.strip()
            if item_playtime.endswith('万'):
                item_playtime=float(item_playtime[:-1])*1000
            item_playtime=int(item_playtime)
            item_subtitle = item.find(class_='so-icon hide').text.strip()
            if item_subtitle.endswith('万'):
                item_subtitle=float(item_subtitle[:-1])*1000
            item_subtitle=int(item_subtitle)
            item_time = item.find(class_='so-icon time').text.strip()
            item_up = item.find(class_='up-name').text

            print("读取 | " + item_title)
            nonlocal n

            sheet.write(n, 0, item_title)
            sheet.write(n, 1, item_link)
            sheet.write(n, 2, item_des)
            sheet.write(n, 3, item_playtime)
            sheet.write(n, 4, item_subtitle)
            sheet.write(n, 5, item_time)
            sheet.write(n, 6, item_up)

            n += 1

    def next_page(des_page):
        try:
            print('读取下一页...')
            next_btn = WAIT.until(EC.element_to_be_clickable((By.CSS_SELECTOR,
                                                              '#all-list > div.flow-loader > div.page-wrap > div > ul > li.page-item.next > button')))
            next_btn.click()
            WAIT.until(EC.text_to_be_present_in_element((By.CSS_SELECTOR,
                                                         '#all-list > div.flow-loader > div.page-wrap > div > ul > li.page-item.active > button'),
                                                        str(des_page)))
            getPage()
        except TimeoutException:
            print('访问超时，尝试刷新中...')
            browser.refresh()
            next_page(des_page)

    total = search()

    for i in range(2, total + 1):
        next_page(i)

    browser.close()

    # 保存表格(仅用于表格存储时)
    book.save(src+'.xls')

if __name__ =='__main__':
    src=input("请输入要搜索的内容：")
    searchB(src)
