from time import sleep 

from selenium.webdriver.common.desired_capabilities import DesiredCapabilities
from selenium import webdriver 
from selenium.webdriver.common.by import By
from selenium.webdriver.support.relative_locator import locate_with
from selenium.webdriver.chrome.options import Options

from bs4 import BeautifulSoup
import lxml
import requests
# import time

import json
import datetime

from xml.etree import ElementTree as ET
import copy

import os
import re

sleep(5)

driver = webdriver.Remote('http://selenium:4444/wd/hub',
                            desired_capabilities = DesiredCapabilities.CHROME)

# driver.get('https://yandex.ru/')
driver.get("https://religion.ranepa.ru/home/archive/2022/620365/")
# driver.save_screenshot('screenshot.png')
sleep(5)
turn_english = [i for i in driver.find_elements(By.CLASS_NAME, 'carte__language') if i.text == 'En'][0]
turn_english.click()
driver.save_screenshot('screenshot_5sec.png')

# Получаем html код из Selenium и адаптируем под BS4. Чтобы если что в Google Colab было проще залить
bigSoup = BeautifulSoup(driver.page_source, 'lxml')


def get_article_info(article_html_part) -> dict:
    """ Функция принимает на вход кусочек html одной статьи.
        Вытаскивает из этого html кусочка данные.
        Формирует словарь. 
        Потом это пригодиться чтобы сформировать xml файл."""

    # Issue №
    try:
        issue_number = bigSoup.find('a', 'archive__subhead-link active').text
    except:
        issue_number = 'issue number has not found'

    # Year
    try:
        year = re.search(r'20\d\d', initial_url)[0]
    except:
        year = 'what is year'

    # soup = BeautifulSoup(article_html_part.get_attribute('outerHTML'), 'lxml')
    soup = article_html_part

    # Articles Title
    try:
        article_title = soup.find('div', 'name').text.strip()
        # Добавить здесь преоброзование строки unicode (Add here trnasformer / encoder from uncode to assii)
        # Но xml не ломается из-за этого и unicode отоброжает корректно. Поэтому пока не вставил
    except:
        article_title = ''

    # Authors names
    try:
        # article_author_name = article.find_element(By.CLASS_NAME, 'author').text
        article_author_name = soup.find('div', 'author').text.strip()
    except:
        article_author_name = ''

    # Key Words
    try:
        # keyword_title = soup.find('div', 'archive__row').find('b', text=re.compile("Keywords"))
        KeyWordss = soup.find('b', text=re.compile("Keywords")).find_next('i')
        KeyWordss = KeyWordss.text.strip()
    except:
        KeyWordss = ''

    # DOI: ****-*****-*****-****
    try:
        doi_parse = soup.find('b', text=re.compile("DOI:"))
        doi_parse = doi_parse.text.strip()
    except:
        doi_parse = ''

    # Section: Main or somthing else
    try:
        # section_title = soup.find('div', 'archive__row').find('b', text=re.compile("Section:"))
        section_text = soup.find(
            'b', text=re.compile("Section:")).find_next('i')
        section_text = section_text.text.strip()
    except:
        section_text = ''

    # Topic
    try:
        # topic_title = soup.find('div', 'archive__row').find('b', text=re.compile("Section:"))
        topic_title = soup.find('b', text=re.compile("Topic:")).find_next('i')
        topic_title = topic_title.text.strip()
    except:
        topic_title = ''

    # Pages:
    try:
        # pages_title = soup.find('div', 'archive__row').find('b', text=re.compile("Pages:"))
        pages_text = soup.find('b', text=re.compile("Pages:")).next_sibling
        pages_text = pages_text.text.strip()
    except:
        pages_text = ''

    # Abstract
    try:
        # absttrect_title = soup.find('div', 'archive__row').find('b', text=re.compile("Abstract"))
        abstract_text = ''.join([i for i in soup.find('b', text="Abstract").find_next(
            'br').find_next('br').next_sibling]).strip()
        # Тэги на некоторые старницых криво стоят, и чтобы аннотация парсилась целиком нужно
        if abstract_text[-1] != '.':
            abstract_text = abstract_text + ''.join([i for i in soup.find(
                'b', text="Abstract").find_next('br').find_next('br').find_next('br').next_sibling]).strip()

        if abstract_text == '-':
            abstract_text = 'N/A'
    except:
        abstract_text = 'N/A'

    # lINKS pdf. Dict of articles PDF's Links. Ru + En if it exists
    try:
        # article_pdf_ru_link = article.find_elements(By.CLASS_NAME, 'lang')
        # article_pdf_links = { i.text : i.get_attribute('href') for j in article_pdf_ru_link for i in j.find_elements(By.CSS_SELECTOR, 'a')}
        article_pdf_links = {link.text.strip(): 'https://religion.ranepa.ru' +
                             link['href'] for link in soup.find('div', 'lang').find_all('a', href=True)}

    except:
        print('Кажется не нашел ссылку на какую-то статью')
        article_pdf_links = ''

    global current_iter
    current_iter += 1

    # Создаем словарь в котором будут хранитьсч вот эти данные.
    article_data_dicts = {
        # Номер статьи которая идет по счету на web странице и в общем зачете. от 1 до 687
        'number': current_iter,
        'Issue number': issue_number,
        'Year': year,
        'Article Title': article_title,
        'Author': article_author_name,
        'KeyWords': KeyWordss,
        # Так как BS4 находит тэг целиком с словом и цифрами типо такого - DOI: 987134-890-348950, то букаы нам не нужны. Обрубаем их
        'DOI': doi_parse[4:].strip(),
        'Section': section_text,
        'Topic': topic_title,
        'Pages': pages_text,
        'Abstract': abstract_text,
        'Links': article_pdf_links,
    }
    article_data_dict = copy.deepcopy(article_data_dicts)
    return article_data_dict


def from_article_info_to_dict_authors(article_info_dict_1: dict) -> list:
    """ Функция формирует список словарей с инофрмацией об авторах на основе article_info (ранее собранных данных). 
        Информация об авторах - это (имя, фамилия, email).
        Пока что email у всех пустой.
        
        Функция возвращает список словарей с информацией об авторах"""

    dict_authors = []
    for key, value in article_info_dict_1.items():
        if key == 'Author':
            if value == ' ' or value == '' or len(value.split(', ')) == 0:
                print('None')
                name_data = {
                    'name': '',
                    'surname': '',
                    'email': '',
                }
                dict_authors.append(name_data)
            elif len(value.split(', ')) >= 2:
                # print('Bigger 2')
                name_data = []
                name_1 = value.split(', ')
                name_2 = {}
                number_of_author = 1
                for name_surname in name_1:
                    # print(name_surname)
                    name_2['name'] = ' '.join(
                        name_surname.strip().split(' ')[0:-1])
                    name_2['surname'] = name_surname.strip().split(' ')[-1]
                    name_2['email'] = ''
                    name_3 = copy.deepcopy(name_2)
                    # name_data[str(number_of_author)] = name_2
                    number_of_author += 1
                    dict_authors.append(name_3)
                    # print(name_2)
            elif len(value.split(', ')) == 1:
                # print('One')
                name_data = {}
                name_2 = {}
                name_2['name'] = ' '.join(value.strip().split(' ')[0:-1])
                name_2['surname'] = value.strip().split(' ')[-1]
                name_2['email'] = ''
                dict_authors.append(name_2)
            elif len(value.split(', ')) == 0:
                # print('None')
                name_data = {}
                name_2 = {}
                name_2['name'] = ''
                name_2['surname'] = value[:]
                name_2['email'] = ''
                dict_authors.append(name_2)
    dict_authors = copy.deepcopy(dict_authors)
    return dict_authors


if not os.path.exists(os.path.dirname('xml/')):
    os.makedirs('xml')


def generate_taplate_for_xml_file(article_info_dict_1: dict) -> None:

    dict_authors = copy.deepcopy(
        from_article_info_to_dict_authors(article_info_dict_1))
    # Функция from_article_info_to_dict_authors() писали до этого. Возврашает список словарей с инфой по авторам.

    root = ET.Element(
        'article', xmlns_xlink="http://www.w3.org/1999/xlink", dtd_version="1.1")  # [1] [1]
    front = ET.SubElement(root, 'front')

    journal_meta = ET.SubElement(front, 'journal-meta')
    journal_id = ET.SubElement(
        journal_meta, 'journal-id', journal_id_type="publisher")  # [1]
    journal_id.text = 'GRC'  # Здесь пишем название издательства

    journal_title_group = ET.SubElement(journal_meta, 'journal-title-group')
    journal_title = ET.SubElement(journal_title_group, 'journal-title')
    # Здесь пишем название Журнала.
    journal_title.text = "Gosudarstvo, religiia, tserkov' v Rossii i za rubezhom "

    # [1] | Что-то особенное нужно вставить в abbrev-type?
    abbrev_journal_title = ET.SubElement(
        journal_title_group, 'abbrev-journal-title', abbrev_type='nlm-ta')

    isnn_ppub = ET.SubElement(journal_meta, 'issn', pub_type="ppub")
    isnn_epub = ET.SubElement(journal_meta, 'issn', pub_type="epub")

    article_meta = ET.SubElement(front, 'article-meta')
    article_id = ET.SubElement(
        article_meta, 'article-id', pub_id_type='manuscript')  # [1]
    article_id.text = ''  # Какой код сюда вставить?

    article_categories = ET.SubElement(article_meta, 'article-categories')
    subj_group_article_type = ET.SubElement(
        article_categories, 'subj-group', subj_group_type="article_type")  # [1]
    subject = ET.SubElement(subj_group_article_type, 'subject')
    subject.text = 'Research Letter'  # Что сюда вставиьт ?

    subj_group_subject_areas = ET.SubElement(
        article_categories, 'subj-group', subj_group_type="subject_areas")  # [1]
    # ['area1', 'area2'] # для теста. сюда вставить данные по subject areas если будет
    subject_areas = ' '
    for word in subject_areas:
        subject = ET.SubElement(subj_group_subject_areas, 'subject')
        subject.text = word  # Вставить всюда subject_areas

    title_group = ET.SubElement(article_meta, 'title-group')
    article_title = ET.SubElement(title_group, 'article-title')
    # 'Название статьи'
    article_title.text = article_info_dict_1['Article Title']

    alt_title = ET.SubElement(title_group, 'alt-title',
                              alt_title_type="running")  # [1]
    alt_title.text = ''  # Вставить какое-то alt название ???

    contrib_group = ET.SubElement(article_meta, 'contrib-group')
    # dict_authors = [
    # 	{'name': "Aleksey",
    # 	'surname': 'Kalinov',
    # 	'email': '',
    # 	},
    # 	{'name': "Petr",
    # 	'surname': 'Vorlamov',
    # 	'email': '',
    # 	}
    # ]
    for author in dict_authors:
        # print(author['name'],
        #     author['surname'],
        #     dict_authors.index(author)+1)
        contrib = ET.SubElement(contrib_group, 'contrib',
                                contrib_type="author")  # [1]
        name_xml = ET.SubElement(contrib, 'name')

        try:
            surname_xml = ET.SubElement(name_xml, 'surname')
            surname_xml.text = str(author['surname'])
        except:
            pass

        try:
            given_names = ET.SubElement(name_xml, 'given-names')
            given_names.text = str(author['name'])
        except:
            pass

        try:
            email_xml = ET.SubElement(contrib, 'email')
            email_xml.text = str(author['email'])
        except:
            pass
        role_contrib = ET.SubElement(contrib, 'role', content_type=str(
            dict_authors.index(author)+1))  # [1]
        aff_number = 'aff' + str(dict_authors.index(author)+1)
        xref = ET.SubElement(contrib, 'xref', ref_type='aff', rid=aff_number)

    for author in dict_authors:
        aff_number = 'aff' + str(dict_authors.index(author)+1)
        aff = ET.SubElement(contrib_group, 'aff', id=aff_number)
        institution = ET.SubElement(aff, 'institution')
        if 'institution' in author:
            # Где взять институт ? Указать Ранхигс или нужен универ атвора?
            institution.text = author['institution']

        addrline1 = ET.SubElement(
            aff, 'addr-line', content_type="addrline1")  # [1]
        if 'addrline1' in author:
            # Где взять адресс ? Указать адресс универа?
            addrline1.text = author['addrline1']

        city_xml = ET.SubElement(aff, 'addr-line', content_type='city')
        if 'city' in author:
            city_xml.text = author['city']

        state_xml = ET.SubElement(aff, 'addr-line', content_type='state')
        if 'state' in author:
            state_xml.text = author['state']

        zipcode_xml = ET.SubElement(aff, 'addr-line', content_type='zipcode')
        if 'zipcode' in author:
            zipcode_xml.text = author['zipcode']

        country_xml = ET.SubElement(aff, 'addr-line', content_type='country')
        if 'country' in author:
            city_xml.text = author['country']

    if 'coresp' in author:
        autor_notes = ET.SubElement(article_meta, 'author-notes')
        coresp_id = ET.SubElement(autor_notes, id='cor1')
        label_coresp = ET.SubElement(coresp_id, 'label')
        label_coresp.text = '*'
        bold_coresp = ET.SubElement(coresp_id, 'bold')
        bold_coresp.text = 'Corresponding Author'
        after_bold = ET.SubElement(coresp_id)
        after_bold.text = ', '.replace(
            map(str, [value for key, value in author.items()]))

    pub_date_epub = ET.SubElement(
        article_meta, 'pub-date', pub_type='epub')  # [1]
    pub_date_ppub = ET.SubElement(
        article_meta, 'pub-date', pub_type='ppub')  # [1]

    elocation_id = ET.SubElement(article_meta, 'elocation-id')
    # Примудать от куда взять это ID он такой же как и в article_id.text
    elocation_id.text = article_id.text

    history = ET.SubElement(article_meta, 'history')
    date_xml = ET.SubElement(history, 'date', data_type='received')
    day_xml = ET.SubElement(date_xml, 'day')
    day_xml.text = ''  # Где брать дату?
    month_xml = ET.SubElement(date_xml, 'month')
    month_xml.text = ''  # Где брать дату?
    year_xml = ET.SubElement(date_xml, 'year')
    # year_xml.text = '' # Где брать дату? | Год взял из парсенного журанала
    year_xml.text = article_info_dict_1['Year']

    permission_xml = ET.SubElement(article_meta, 'permissions')
    copyright_statement = ET.SubElement(permission_xml, 'copyright-statement')
    copyright_statement.text = ''  # Где брать копирайт выражение? Оно есть у ГРЦ?
    copyright_year = ET.SubElement(permission_xml, 'copyright-year')
    # Где брать год для коипарайта? Или указывать 2022, но не уверен, что это законно
    copyright_year.text = ''

    #Absract
    abstract_xml = ET.SubElement(article_meta, 'abstract')
    abstract_b_xml = ET.SubElement(abstract_xml, 'b')
    if len(article_info_dict_1['Abstract']) > 1:
        abstract_b_xml.text = article_info_dict_1['Abstract']
    else:
        abstract_b_xml.text = 'N/A'

    key_words_group = ET.SubElement(article_meta, 'kwd-group')

    # article_test = {
    # 	'title': 'The numerous artifacts in the sky of peoples mind',
    # 	'key_words': ['word1', 'drugs', 'math', 'comupterScience']}
    try:
        for key_word in article_info_dict_1['KeyWords'].split(', '):
            kwd = ET.SubElement(key_words_group, 'kwd')
            kwd.text = key_word
    except:
        pass

    counts_xml = ET.SubElement(article_meta, 'counts')
    # Считать таблицы или забить ?
    table_count = ET.SubElement(counts_xml, 'table-count', count='0')
    # Вроде нужно считать старницы, но в примере Джэксона стоят 0. Может что-то другое считают, а навазвание совпало. Ну или они подзабивают на точность и аккуратность данных в xml документах
    page_count = ET.SubElement(counts_xml, 'page-count', count='0')

    def prettify(element, indent='  '):
        """ Фцнкция преобразовывает одностроничный xml в красивый многострочный xml с отсупыми"""
        queue = [(0, element)]  # (level, element)
        while queue:
            level, element = queue.pop(0)
            children = [(level + 1, child) for child in list(element)]
            if children:
                element.text = '\n' + indent * (level+1)  # for child open
            if queue:
                element.tail = '\n' + indent * queue[0][0]  # for sibling open
            else:
                element.tail = '\n' + indent * (level-1)  # for parent close
            queue[0:0] = children  # prepend so children come before siblings

    prettify(root)

    tree = ET.ElementTree(root)
    tree.write('xml/sample.xml', encoding='UTF-8', xml_declaration=True)


def cleanning_and_save_xml_file(article_info_dict_2: dict,
                                num_of_article: int | str,
                                num_of_issue: int | str) -> None:
    """ Функция очищает файл Sample.xml от ломающих xml кодов.

        Функция дорабаватывает xml файлов, там где '_' ставит '-' (Сразу не получилось поставить из-за особенностей Python)

        Функция сохраняет чистые и красивые  xml файлы под порядковым номером статьи. (номер формируется автоматически) 

        Функция формирует папку с ошибьками и файл с описанием ошибки. 
        Если во время сохранения возникли ошибки, то их тоже сохранет в папку erorrs. 
        Если ошибок не было, то и папки erorrs нет"""

    number_of_current_articles_counter = article_info_dict_2['number']
    year_xmll = article_info_dict_2['Year']
    # num_of_issue = article_info_dict_2['Issue number']
    # Read in the file
    with open('xml/sample.xml', 'r', encoding='UTF-8') as file:
        filedata = file.read()

    for_replace_in_xml = {
        "xmlns_xlink": 'xmlns:xlink',
        "dtd_version": "dtd-version",
        "ournal_id_type": 'journal-id-type',
        'abbrev_type': 'abbrev-type',
        'pub_id_type': 'pub-id-type',
        'subj_group_type': 'subj-group-type',
        'subj_group_type': 'subj-group-type',
        'alt_title_type': 'alt-title-type',
        'contrib_type': 'contrib-type',
        'content_type': 'content-type',  # Не 100% будет
        'pub_type': 'pub-type',
        'pub_type': 'pub-type',
        'data_type': 'data-type',
        'ref_type': 'ref-type',
        '': '"',  # из-за этой истории крашиться xml документ
    }

    for wrong, right in for_replace_in_xml.items():
        filedata = filedata.replace(wrong, right)
    # # Заменяем косячные слова на правильные с дефисом
    # filedata = filedata.replace('dtd_version', 'dtd-version')

    # Write the file out again
    # path_for_saving_lxml = f'lxml/lxml_{number_of_current_articles_counter}.xml'

    filename_1 = f'xml/{year_xmll}/{num_of_issue}/'

    if len(article_info_dict_2['Author']) <= 0 and len(article_info_dict_2['Abstract']) <= 3:

        filename_erroes = f'xml/{year_xmll}/{num_of_issue}/errors/'
        path_for_saving_lxml = f'xml/{year_xmll}/{num_of_issue}/errors/{num_of_article}_lxml_{number_of_current_articles_counter}.xml'

        # Формируем txt файл где пытаемся сказать что не так и дать какую-то отладочную информацию
        dict_that_contains_data_from_article_info = {
            "article_info_dict_2['Article Title']": len(article_info_dict_2['Article Title']),
            "article_info_dict_2['Author']": len(article_info_dict_2['Author']),
            "article_info_dict_2['Links']": len(article_info_dict_2['Links']),
            "article_info_dict_2['KeyWords']": len(article_info_dict_2['KeyWords']),
            "article_info_dict_2['DOI']": len(article_info_dict_2['DOI']),
            "article_info_dict_2['Section']": len(article_info_dict_2['Section']),
            "article_info_dict_2['Topic']": len(article_info_dict_2['Topic']),
            "article_info_dict_2['Pages']": len(article_info_dict_2['Pages']),
            "article_info_dict_2['Abstract']": len(article_info_dict_2['Abstract']),
            "article_info_dict_2['Year']": len(article_info_dict_2['Year']),
        }
        filedata_erorrs = []
        filedata_erorrs_2 = []
        # Формируем файл для отладки. Чтобы было проще искать где файл
        filedata_erorrs.append(
            f'Что-то пошло не так.{path_for_saving_lxml} \nГод: {year_xmll} \nВыпуск №: {num_of_issue}\nСтатья в выпуске №: {num_of_article}\nСтатья в общем подсчете№: {number_of_current_articles_counter} \n\n\n')

        # Формируем данные что именно выглядит подозриетльно
        for article_info_key, len_article_info_value in dict_that_contains_data_from_article_info.items():
            if len_article_info_value <= 1:
                filedata_erorrs.append(
                    f'Не найдено (или подозрительно короткое):{article_info_key[19:]}\n')

            dict_that_contains_data_from_article_info_2 = {
                "article_info_dict_2['Article Title']": article_info_dict_2['Article Title'],
                "article_info_dict_2['Author']": article_info_dict_2['Author'],
                "article_info_dict_2['Links']": article_info_dict_2['Links'],
                "article_info_dict_2['KeyWords']": article_info_dict_2['KeyWords'],
                "article_info_dict_2['DOI']": article_info_dict_2['DOI'],
                "article_info_dict_2['Section']": article_info_dict_2['Section'],
                "article_info_dict_2['Topic']": article_info_dict_2['Topic'],
                "article_info_dict_2['Pages']": article_info_dict_2['Pages'],
                "article_info_dict_2['Abstract']": article_info_dict_2['Abstract'],
                "article_info_dict_2['Year']": article_info_dict_2['Year'],
            }

            for article_info_key_2, len_article_info_value_2 in dict_that_contains_data_from_article_info_2 .items():
                if len_article_info_value <= 1:
                    filedata_erorrs_2.append(
                        f'{article_info_key_2[19:]}: ' + str(len_article_info_value_2) + '\n')

        filedata_erorrs.append('\n\nЧто храниться в этой дате\n\n')
        filedata_erorrs.append(' '.join(set(filedata_erorrs_2)))

        filedata_erorrs = ' '.join(filedata_erorrs)

        path_for_saving_txt = f'xml/{year_xmll}/{num_of_issue}/errors/{num_of_article}_lxml_{number_of_current_articles_counter}.txt'
        if not os.path.exists(os.path.dirname(filename_erroes)):
            os.makedirs(filename_erroes)

        with open(path_for_saving_lxml, 'w', encoding='UTF-8') as file:
            file.write(filedata)

        with open(path_for_saving_txt, 'w', encoding='UTF-8') as file:
            file.write(filedata_erorrs)
    else:
        if not os.path.exists(os.path.dirname(filename_1)):
            os.makedirs(filename_1)
        path_for_saving_lxml = f'xml/{year_xmll}/{num_of_issue}/{num_of_article}_lxml_{number_of_current_articles_counter}.xml'

        with open(path_for_saving_lxml, 'w', encoding='UTF-8') as file:
            file.write(filedata)
    return 'hello saving'


base_url = 'https://religion.ranepa.ru/'

# Находит ссылки на архив каждого года
archive_all_years = {year_link.text: base_url + year_link['href'] for year_link in bigSoup.find_all('a', 'sidebar__year-number', href=True)}

#  Находим ссылки на выпуски
issues_links = {issue.text: base_url + issue['href'] for issue in bigSoup.find_all('a', 'archive__subhead-link') if issue.text != 'Download all'}



current_iter = 0

for year_text, year_link in archive_all_years.items():
    sleep(1.5)
    initial_url = year_link
    driver.get(initial_url)

    bigSoup = BeautifulSoup(driver.page_source, 'lxml')
    issues_links = {issue.text: base_url + issue['href'] for issue in bigSoup.find_all(
        'a', 'archive__subhead-link')if issue.text != 'Download all'}

    for text, link in issues_links.items():

        sleep(1)
        initial_url = link
        driver.get(initial_url)

        bigSoup = BeautifulSoup(driver.page_source, 'lxml')

        num_of_article = 1

        for article in bigSoup.find_all('div', 'archive__row'):
            article_info = copy.deepcopy(get_article_info(article))
            generate_taplate_for_xml_file(article_info)
            cleanning_and_save_xml_file(
                article_info, num_of_article=num_of_article, num_of_issue=text)
            num_of_article += 1

print(f"Привет\n Эта программа помогает собирать и преоброзовывать данные с сайта журнала: Госудраство Религия Церьковь")
print('Программа создана чтобы собрать и преоброзовать все статьи за все года и все выпуски \n По времени займет примерно 3-5 минут. Возможно быстрее')
print('Во время работы программы, здесь могут появляться слова None. \nЕсли такое случитсья не переживайте, все хорошго, так и должно быть')

print(f'Все спарсилось, смотритве в папке:\n {os.getcwd() + "/xml"}')
