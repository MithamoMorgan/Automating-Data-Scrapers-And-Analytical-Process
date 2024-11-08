## Overview

This project aims to scrape Flash Sales products from Jumia website and automate the scripts and analytical process using Apache Airflow. This project focuses on workflow automation and scheduling. This project uses docker to run Airflow.

## Tools

* **Language:** Python
* **Orchestration Tool:** Apache Airflow
* **Libraries:** Pandas, Numpy, Beautiful Soup
* **Containerization Platform** Docker (Used to run Airflow services like webserver, scheduler, and worker in isolated containers)

## Code

```python
import os
import pandas as pd
from bs4 import BeautifulSoup
import requests
from urllib.parse import urljoin
from airflow import DAG
from datetime import datetime, timedelta
from airflow.operators.python_operator import PythonOperator

default_args = {
    'owner':'devscraper',
    'retries':2,
    'retry_delay':timedelta(minutes = 1)
}

def scrape():

    # Create directory if it doesn't exist
    dir_path = '/opt/airflow/airflow_data/csv'
    os.makedirs(dir_path, exist_ok=True)

    url = 'https://www.jumia.co.ke/flash-sales/' 
    details_list = []
    
    while True:
        response = requests.get(url)
        soup = BeautifulSoup(response.text, 'html.parser')
        
        details = soup.find_all('div', class_ = 'info')
        footers  = soup.find_all('footer', class_ = 'ft')
    
        for detail,footer in zip(details,footers):
            name = detail.find('h3', class_ = 'name').text.strip()
            price =  detail.find('div', class_ = 'prc').text.strip()
            try:
                old_price = detail.find('div', class_ = 'old').text.strip()
            except:
                old_price = " "
            try:
                discount = detail.find('div', class_ = 'bdg _dsct _sm').text.strip()
            except:
                discount = " "
            try:
                stars_reviews = detail.find('div', class_ = 'rev').text.strip()
            except:
                stars_reviews = " "
        
            try:
                items_left = footer.find('div', class_ = 'stk').text.strip()
            except:
                items_left = " Out of stock"
                
                
        
            details_dict = {'Name':name,'price':price,'Old Price':old_price,'discount':discount,'Stars and Reviews':stars_reviews,'Items Left':items_left}
            details_list.append(details_dict)
    
        next_page = soup.find('a',{'class':'pg', 'aria-label':'Next Page'})
        if next_page:
            next_url = next_page.get('href')
            url = urljoin(url,next_url)
        else:
            break
    
    flash_sale_data = pd.DataFrame(details_list)
    # save in a csv file
    flash_sale_data.to_csv(os.path.join(dir_path, 'jumia_df.csv'), index=False)

def clean_data():
    dir_path = '/opt/airflow/airflow_data/csv'
    flash_sale_data = pd.read_csv(os.path.join(dir_path, 'jumia_df.csv'))

    # Change numeric columns to numeric types
    flash_sale_data['price'] = flash_sale_data['price'].str.replace(r'[^\d.]', '', regex=True).astype('float')
    flash_sale_data['Old Price'] = flash_sale_data['Old Price'].str.replace(r'[^\d.]', '', regex=True)
    flash_sale_data['Old Price'] = pd.to_numeric(flash_sale_data['Old Price'], errors='coerce')
    flash_sale_data['discount'] = flash_sale_data['discount'].str.replace(r'[^\d.]', '', regex=True)
    flash_sale_data['discount'] = pd.to_numeric(flash_sale_data['discount'], errors='coerce')
    flash_sale_data['Items Left'] = flash_sale_data['Items Left'].str.replace(r'[^\d.]', '', regex=True)
    flash_sale_data['Items Left'] = pd.to_numeric(flash_sale_data['Items Left'], errors = 'coerce')


    # Extract Stars and Reviews from Stars and Reviews column
    flash_sale_data[['Stars', 'Reviews']] = flash_sale_data['Stars and Reviews'].str.extract(r'([\d.]+) out of 5\((\d+)\)')
    flash_sale_data['Stars'] = pd.to_numeric(flash_sale_data['Stars'], errors='coerce')
    flash_sale_data['Reviews'] = pd.to_numeric(flash_sale_data['Reviews'], errors='coerce')
    
    # Drop the column Stars and Reviews
    flash_sale_data = flash_sale_data.drop('Stars and Reviews', axis=1)

    # Save the cleaned data back to CSV
    flash_sale_data.to_csv(os.path.join(dir_path, 'cleaned_jumia_df.csv'), index=False)


with DAG(
    dag_id = 'jumia_scraping_dag',
    default_args = default_args,
    description = 'Writing my first dag to scrape data',
    start_date = datetime(2024,10,31, 11),
    schedule_interval = '@daily'
) as dag :

    scrape_task = PythonOperator(
        task_id = 'scrape',
        python_callable = scrape
    )

    clean_task = PythonOperator(
    task_id='clean_data',
    python_callable=clean_data
)

scrape_task >> clean_task
```

* The Airflow Dag currently has 2 tasks that is the cleaning task and the scraping tasks. The Code is scheduled to run daily hence getting upto date data from Jumia website.
  
