---
layout: single
title:  "WebScraping"
categories: Programming
tag: [Python, Web]
toc: true
---

# Web_Scraping


### 소개

*BeautifulSoup*를 이용한 웹스크래핑 코드입니다.



### CODE

```py
from bs4 import BeautifulSoup
from requests import get


base_url = "https://weworkremotely.com/remote-jobs/search?term="

search_term = "python"

## search_term 으로 검색되는 base_url 주소를 get 함수에서 리턴된 http 코드를 response에 저장
response = get(f"{base_url}{search_term}")

## response의 status_code가 200(정상)이 아니면 오류 메시지 출력

if response.status_code != 200:
    print("Can't request website")

## get으로 리턴된 값이 200(정상) 일 경우

else:
    results = []##결과값을 for문 밖에 저장하기 위해 만든 리스트 변수
    soup = BeautifulSoup(response.text, "html.parser")
    jobs = soup.find_all('section', class_='jobs')
    ## jobs클래스를 가진 section이 여러개이기 때문에 for문을 돌린다
    
    for job_section in jobs:
        job_posts = job_section.find_all('li')

        ## jobs_posts의 마지막은 section 안에 있는 li 이긴 하지만 class가 view_all인 버튼이다
        ##따라서 .pop을 이용해 마지막 값을 지운다.

        job_posts.pop(-1)
        for post in job_posts:

           ## li 안에 a들을 anchors로 저장하고 li의 첫번째 a는 단순한 로고 이미지기 떄문에
           ## 또 다른 변수 anchor에다가 두번째 a를 저장

           anchors = post.find_all('a')
           anchor = anchors[1]
           link = (anchor['href'])
           company, kind, region = anchor.find_all('span', class_="company")
           title = anchor.find('span', class_='title')
           job_data = {
               'company':company.string,
               'region':region.string,
               'position':title.string,
           }
           results.append(job_data)
    
    for result in results:
        print(result)
        print("/////////////////////")
```



### 마무리

좀 더 사용하기 좋고 보기 편한 인터페이스로 발전이 필요합니다.
