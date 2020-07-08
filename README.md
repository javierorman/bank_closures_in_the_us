# Bank Closures in the US: 2000-2020

##### Javier Orman
(Updated 5-22-2020)

## Introduction

The **2008 financial crisis** in the United States saw the closing of numerous banks, notably some large ones (Washington Mutual Bank being the largest with $307b in assets) and many small ones. 
As a new student of programming and data science, I decided to learn and practice a few important skills by **gathering, cleaning, organizing and visualizing data** on the bank closures that have taken place in the United States over the last 20 years. The visualizations in particular will lead to important and interesting questions.

The main source data is the **FDIC Failed Bank List**. It is available to the public [here](https://www.fdic.gov/bank/individual/failed/banklist.csv). Initially, I located the data through [Data.gov](https://catalog.data.gov/dataset/fdic-failed-bank-list-9889f/resource/d8ca97d4-87e7-4d89-8922-d266b7b07d52).

The csv contains information on all bank closures in the United States since October 1, 2000.

[A second data source](https://www.fdic.gov/bank/historical/bank/index.html), also from the FDIC, will be used in Chapter 5.

## Index

[1. Retrieving the data](#section_1)

[2. Cleaning the data](#section_2)

[3. Bank closures by date](#section_3)

[4. Bank closures by state](#section_4)

[5. Bank closures by assets](#section_5)

[6. Acquiring institutions](#section_6)

[7. Additional resources](#section_7)

<a id=’section_1’></a>

### 1. Retrieving the data

We'll start by retrieving the data from the csv file and organizing it in SQL tables


```python
import csv
import psycopg2
import pandas as pd
```


```python
# Print a few lines from the source csv file

with open('banklist.csv', newline='') as f:
    reader = csv.reader(f)
    i = 0
    for row in reader:
        if i < 6:
            print(row)
            i += 1
        else: break
```

    ['Bank Name', 'City', 'ST', 'CERT', 'Acquiring Institution', 'Closing Date']
    ['Old Harbor Bank', 'Clearwater', 'FL', '57537', '1st United Bank', '21-Oct-11']
    ['The Bank of Miami,N.A.', 'Coral Gables', 'FL', '19040', '1st United Bank', '17-Dec-10']
    ['Republic Federal Bank, N.A.', 'Miami', 'FL', '22846', '1st United Bank', '11-Dec-09']
    ['The Bank of Commerce', 'Wood Dale', 'IL', '34292', 'Advantage National Bank Group', '25-Mar-11']
    ['Prosperan Bank', 'Oakdale', 'MN', '35074', 'Alerus Financial, N.A.', '6-Nov-09']



```python
# Create and connect to database Bank_Closures:

try:
    conn = psycopg2.connect("dbname='Bank_Closures' user='postgres' host='localhost' password='postgres'")
except:
    print("I am unable to connect to the database")

cur = conn.cursor()
```


```python
# Create table bank_list that will host our data:
    
cur.execute("""
    DROP TABLE IF EXISTS bank_list;
    CREATE TABLE bank_list (
                id SERIAL NOT NULL PRIMARY KEY,
                bank_name VARCHAR(150),
                city VARCHAR(50),
                state VARCHAR(2),
                cert INT,
                acqinst VARCHAR(100),
                date_of_closure VARCHAR(10)
           );
""")
```


```python
# Insert each row from the csv file into table bank_list:

with open('banklist.csv', 'r') as f:
    reader = csv.reader(f)
    next(reader) # Skip the header row.
    for row in reader:
        cur.execute(
        "INSERT INTO bank_list (bank_name, city, state, cert, acqinst, date_of_closure) VALUES (%s, %s, %s, %s, %s, %s)",
        row
    )
```


```python
# Print a few rows from the table

cur.execute("SELECT * FROM bank_list LIMIT 5")
sample = cur.fetchall()
for row in sample:
    print(row)
```

    (1, 'Old Harbor Bank', 'Clearwater', 'FL', 57537, '1st United Bank', '21-Oct-11')
    (2, 'The Bank of Miami,N.A.', 'Coral Gables', 'FL', 19040, '1st United Bank', '17-Dec-10')
    (3, 'Republic Federal Bank, N.A.', 'Miami', 'FL', 22846, '1st United Bank', '11-Dec-09')
    (4, 'The Bank of Commerce', 'Wood Dale', 'IL', 34292, 'Advantage National Bank Group', '25-Mar-11')
    (5, 'Prosperan Bank', 'Oakdale', 'MN', 35074, 'Alerus Financial, N.A.', '6-Nov-09')


<a id=’section_2’></a>

### 2. Cleaning the data

As we can see, the dates in our date_of_closure column have a **'DD-Mon-YY' format**. In order to avoid issues later on, I will convert (alter) the whole column using Postgres function **to_date**.


```python
cur.execute("""ALTER TABLE bank_list ALTER COLUMN date_of_closure TYPE DATE 
            using to_date(date_of_closure, 'DD-Mon-YY');""")
conn.commit()
```

See our data so far by transfering from **table_list** to a pandas dataframe, using **SQLAlchemy**.


```python
from sqlalchemy import create_engine

engine = create_engine('postgresql://postgres:postgres@localhost:5432/Bank_Closures')
alconn = engine.raw_connection()
alcur = alconn.cursor()
```


```python
df = pd.read_sql_table("bank_list", engine)
```

Our dataframe, with clean dates in **date_of_closure** column:


```python
df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>bank_name</th>
      <th>city</th>
      <th>state</th>
      <th>cert</th>
      <th>acqinst</th>
      <th>date_of_closure</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>Old Harbor Bank</td>
      <td>Clearwater</td>
      <td>FL</td>
      <td>57537</td>
      <td>1st United Bank</td>
      <td>2011-10-21</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>The Bank of Miami,N.A.</td>
      <td>Coral Gables</td>
      <td>FL</td>
      <td>19040</td>
      <td>1st United Bank</td>
      <td>2010-12-17</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>Republic Federal Bank, N.A.</td>
      <td>Miami</td>
      <td>FL</td>
      <td>22846</td>
      <td>1st United Bank</td>
      <td>2009-12-11</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>The Bank of Commerce</td>
      <td>Wood Dale</td>
      <td>IL</td>
      <td>34292</td>
      <td>Advantage National Bank Group</td>
      <td>2011-03-25</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>Prosperan Bank</td>
      <td>Oakdale</td>
      <td>MN</td>
      <td>35074</td>
      <td>Alerus Financial, N.A.</td>
      <td>2009-11-06</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>556</th>
      <td>557</td>
      <td>City Bank</td>
      <td>Lynnwood</td>
      <td>WA</td>
      <td>21521</td>
      <td>Whidbey Island Bank</td>
      <td>2010-04-16</td>
    </tr>
    <tr>
      <th>557</th>
      <td>558</td>
      <td>First NBC Bank</td>
      <td>New Orleans</td>
      <td>LA</td>
      <td>58302</td>
      <td>Whitney Bank</td>
      <td>2017-04-28</td>
    </tr>
    <tr>
      <th>558</th>
      <td>559</td>
      <td>Mirae Bank</td>
      <td>Los Angeles</td>
      <td>CA</td>
      <td>57332</td>
      <td>Wilshire State Bank</td>
      <td>2009-06-26</td>
    </tr>
    <tr>
      <th>559</th>
      <td>560</td>
      <td>Virginia Business Bank</td>
      <td>Richmond</td>
      <td>VA</td>
      <td>58283</td>
      <td>Xenith Bank</td>
      <td>2011-07-29</td>
    </tr>
    <tr>
      <th>560</th>
      <td>561</td>
      <td>First Federal Bank</td>
      <td>Lexington</td>
      <td>KY</td>
      <td>29594</td>
      <td>Your Community Bank</td>
      <td>2013-04-19</td>
    </tr>
  </tbody>
</table>
<p>561 rows × 7 columns</p>
</div>



<a id=’section_3’></a>

### 3. Bank closures by date

Back in the database, set up table of bank closures grouped by **date**


```python
cur.execute("""
    DROP TABLE IF EXISTS Closures_by_Date;
    CREATE TABLE Closures_by_Date (
        date_of_closure DATE,
        closure_count INTEGER
    );
""")
```


```python
cur.execute("""INSERT INTO Closures_by_Date (date_of_closure, closure_count) 
                SELECT bank_list.date_of_closure, 
                COUNT (*) FROM bank_list GROUP BY bank_list.date_of_closure;""")
conn.commit()
```

Print the **5 dates** with the biggest number of bank closures


```python
df = pd.read_sql_table("closures_by_date", engine)
df.sort_values(by=['closure_count'], inplace=True, ascending=False)
df.head().style.hide_index().format({"date_of_closure": lambda t: t.strftime("%Y-%m-%d")}) 
```




<style  type="text/css" >
</style><table id="T_c98edb70_9c73_11ea_8db1_acbc32cefccf" ><thead>    <tr>        <th class="col_heading level0 col0" >date_of_closure</th>        <th class="col_heading level0 col1" >closure_count</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow0_col0" class="data row0 col0" >2009-10-30</td>
                        <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow0_col1" class="data row0 col1" >9</td>
            </tr>
            <tr>
                                <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow1_col0" class="data row1 col0" >2010-04-16</td>
                        <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow1_col1" class="data row1 col1" >8</td>
            </tr>
            <tr>
                                <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow2_col0" class="data row2 col0" >2010-08-20</td>
                        <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow2_col1" class="data row2 col1" >8</td>
            </tr>
            <tr>
                                <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow3_col0" class="data row3 col0" >2009-10-23</td>
                        <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow3_col1" class="data row3 col1" >7</td>
            </tr>
            <tr>
                                <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow4_col0" class="data row4 col0" >2010-07-23</td>
                        <td id="T_c98edb70_9c73_11ea_8db1_acbc32cefccfrow4_col1" class="data row4 col1" >7</td>
            </tr>
    </tbody></table>




```python
#Get ready to visualize bank closures by date with matplotlib

%matplotlib inline
from matplotlib import pyplot as plt
plt.style.use('seaborn-whitegrid')
```

More interesting than bank closures in any particular day are the number of closings in a longer period of time. I will group the closures **by year** using pandas' **resample function**.


```python
df.date_of_closure = pd.to_datetime(df.date_of_closure)
df1 = df.resample('Y', on='date_of_closure').sum()
df1.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>closure_count</th>
    </tr>
    <tr>
      <th>date_of_closure</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2000-12-31</th>
      <td>2</td>
    </tr>
    <tr>
      <th>2001-12-31</th>
      <td>4</td>
    </tr>
    <tr>
      <th>2002-12-31</th>
      <td>11</td>
    </tr>
    <tr>
      <th>2003-12-31</th>
      <td>3</td>
    </tr>
    <tr>
      <th>2004-12-31</th>
      <td>4</td>
    </tr>
  </tbody>
</table>
</div>



**date_of_closure** is now automatically an index column.


```python
plt.figure(figsize=[12,6])
plt.plot(df1.index, df1.closure_count)
plt.show()
```


    
![png](bank_closures_in_the_us_files/bank_closures_in_the_us_35_0.png)
    


As we already knew, but the visualization confirms, **bank closures in the United States spiked sharply around 2008-10**.

<a id=’section_4’></a>

### 4. Bank closures by state

Set up table of bank closures, grouped by **state**


```python
cur.execute("""
    DROP TABLE IF EXISTS Closures_by_State;
    CREATE TABLE Closures_by_State (
        state VARCHAR(2),
        closure_count INTEGER
    );
""")
```


```python
cur.execute("""INSERT INTO Closures_by_State (state, closure_count) 
                SELECT bank_list.state, 
                COUNT (*) FROM bank_list GROUP BY bank_list.state;""")
conn.commit()
```

Print the **5 states** with the biggest number of bank closures


```python
df = pd.read_sql_table("closures_by_state", engine)
df.sort_values(by=['closure_count'], inplace=True, ascending=False)
df.head(10).style.hide_index()
```




<style  type="text/css" >
</style><table id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccf" ><thead>    <tr>        <th class="col_heading level0 col0" >state</th>        <th class="col_heading level0 col1" >closure_count</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow0_col0" class="data row0 col0" >GA</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow0_col1" class="data row0 col1" >93</td>
            </tr>
            <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow1_col0" class="data row1 col0" >FL</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow1_col1" class="data row1 col1" >75</td>
            </tr>
            <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow2_col0" class="data row2 col0" >IL</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow2_col1" class="data row2 col1" >69</td>
            </tr>
            <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow3_col0" class="data row3 col0" >CA</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow3_col1" class="data row3 col1" >41</td>
            </tr>
            <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow4_col0" class="data row4 col0" >MN</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow4_col1" class="data row4 col1" >23</td>
            </tr>
            <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow5_col0" class="data row5 col0" >WA</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow5_col1" class="data row5 col1" >19</td>
            </tr>
            <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow6_col0" class="data row6 col0" >AZ</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow6_col1" class="data row6 col1" >16</td>
            </tr>
            <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow7_col0" class="data row7 col0" >MO</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow7_col1" class="data row7 col1" >16</td>
            </tr>
            <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow8_col0" class="data row8 col0" >MI</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow8_col1" class="data row8 col1" >14</td>
            </tr>
            <tr>
                                <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow9_col0" class="data row9 col0" >TX</td>
                        <td id="T_f6598bb4_9c73_11ea_8db1_acbc32cefccfrow9_col1" class="data row9 col1" >13</td>
            </tr>
    </tbody></table>




```python
plt.figure(figsize=[30,20], dpi=300)
plt.barh(df.state, df.closure_count)
plt.xticks([0,10,20,30,40,50,60,70,80,90])
plt.xticks(fontsize=18)
plt.yticks(fontsize=14)
plt.gca().invert_yaxis()
plt.show()
```


    
![png](bank_closures_in_the_us_files/bank_closures_in_the_us_44_0.png)
    


The number of closures in **Georgia** is surprising. Georgia is only the 8th most populous state in the United States, and yet it has a higher number of bank closures than much bigger states like California or Texas.
Nobel-prize winner Paul Krugman wrote about this in an opinion piece for the New York Times in 2010: https://www.nytimes.com/2010/04/12/opinion/12krugman.html

At the time, Mr. Krugman attributed this to the number of small banks in Georgia that took advantage of a lack of regulations to irresponsibly expand their lending practices:

> *And for all the concern about banks that are too big to fail, Georgia suffered, if anything, from a proliferation of small banks. Actually, the worst offenders in the lending spree tended to be relatively small start-ups that attracted customers by playing to a specific community. Thus Georgian Bank, founded in 2001, catered to the state’s elite, some of whom were entertained on the C.E.O.’s yacht and private jet. Meanwhile, Integrity Bank, founded in 2000, played up its “faith based” business model. It was featured in a 2005 Time magazine article titled “Praying for Profits.” Both banks have now gone bust.*

It's important to note at this point that, so far, we've been tracking only the **number of bank closures** and not total assets. Next, we'll look at the **size of these banks**.

<a id=’section_5’></a>

### 5. Bank closures by assets

We'll start by importing a **second data source**. This csv also came from the FDIC and covers some of the same data as the previous source, with the addition of **assets**. https://www.fdic.gov/bank/historical/bank/index.html


```python
dfbf = pd.read_csv("bfb-data.csv")
dfbf.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Bank Name, City, State</th>
      <th>Press Release (PR)</th>
      <th>Closing Date</th>
      <th>Approx. Asset (Millions)</th>
      <th>Approx. Deposit (Millions)</th>
      <th>Acquirer &amp; Transaction</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>First State Bank, Barboursville, WV</td>
      <td>PR-046-2020</td>
      <td>3-Apr-20</td>
      <td>$152.40</td>
      <td>$139.50</td>
      <td>MVB Bank has agreed to assume all deposits ex...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Ericson State Bank, Ericson, NE</td>
      <td>PR-011-2020</td>
      <td>14-Feb-20</td>
      <td>$100.90</td>
      <td>$95.20</td>
      <td>Farmers and Merchants Bank has agreed to assum...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>City National Bank of New Jersey, Newark, NJ</td>
      <td>PR-101-2019</td>
      <td>1-Nov-19</td>
      <td>$120.60</td>
      <td>$111.20</td>
      <td>Industrial Bank has agreed to assume all depos...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Resolute Bank, Maumee, OH</td>
      <td>PR-097-2019</td>
      <td>25-Oct-19</td>
      <td>$27.10</td>
      <td>$26.20</td>
      <td>Buckeye State Bank has agreed to assume all de...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Louisa Community Bank, Louisa, KY</td>
      <td>PR-096-2019</td>
      <td>25-Oct-19</td>
      <td>$29.70</td>
      <td>$26.50</td>
      <td>Kentucky Farmers Bank Corporation has agreed t...</td>
    </tr>
  </tbody>
</table>
</div>



Let's re-format the "Approx. Asset" column so it's readable as a number. We'll also sort by that column.


```python
dfbf[dfbf.columns[3]] = dfbf[dfbf.columns[3]].replace('[\$,]', '', regex=True).astype(float)
dfbf = dfbf.drop(index=560)
dfbf.sort_values(by=dfbf.columns[3], inplace=True, ascending=False)
dfbf.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Bank Name, City, State</th>
      <th>Press Release (PR)</th>
      <th>Closing Date</th>
      <th>Approx. Asset (Millions)</th>
      <th>Approx. Deposit (Millions)</th>
      <th>Acquirer &amp; Transaction</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>521</th>
      <td>Washington Mutual Bank, Henderson, NV</td>
      <td>PR-85-2008</td>
      <td>25-Sep-08</td>
      <td>307000.0</td>
      <td>$188,000.00</td>
      <td>In a transaction facilitated by the FDIC, JPMo...</td>
    </tr>
    <tr>
      <th>529</th>
      <td>IndyMac Bank, F.S.B., Pasadena, CA</td>
      <td>PR-56-2008</td>
      <td>11-Jul-08</td>
      <td>32010.0</td>
      <td>$19,060.00</td>
      <td>Non-brokered insured deposits and substantiall...</td>
    </tr>
    <tr>
      <th>435</th>
      <td>Colonial Bank, Montgomery, AL</td>
      <td>PR-143-2009</td>
      <td>14-Aug-09</td>
      <td>25000.0</td>
      <td>$20,000.00</td>
      <td>Branch Banking and Trust Company (BB&amp;T), Winst...</td>
    </tr>
    <tr>
      <th>428</th>
      <td>Guaranty Bank, Austin, TX</td>
      <td>PR-150-2009</td>
      <td>21-Aug-09</td>
      <td>13000.0</td>
      <td>$12,000.00</td>
      <td>BBVA Compass, Birmingham, AL has agreed to ass...</td>
    </tr>
    <tr>
      <th>513</th>
      <td>Downey Savings and Loan Association, F.A., New...</td>
      <td>PR-124-2008</td>
      <td>21-Nov-08</td>
      <td>12800.0</td>
      <td>$9,700.00</td>
      <td>In a transaction facilitated by the FDIC, U.S....</td>
    </tr>
  </tbody>
</table>
</div>



There is a large difference between the biggest bank (Washington Mutual) and the rest. It would be interesting to see the distribution between bank closures and assets.
For this we will create a new dataframe *manually*.


```python
b1 = len(dfbf[dfbf.iloc[:, 3] > 50000])
b2 = len(dfbf[(dfbf.iloc[:, 3] < 50000) & (dfbf.iloc[:, 3] > 10000)])
b3 = len(dfbf[(dfbf.iloc[:, 3] < 10000) & (dfbf.iloc[:, 3] > 5000)])
b4 = len(dfbf[(dfbf.iloc[:, 3] < 5000) & (dfbf.iloc[:, 3] > 1000)])
b5 = len(dfbf[(dfbf.iloc[:, 3] < 1000) & (dfbf.iloc[:, 3] > 500)])
b6 = len(dfbf[(dfbf.iloc[:, 3] < 500)])

list_of_lists = [['More than $50b',b1], ['Between \$10b and \$50b',b2], ['Between \$5b and \$10b',b3], 
                 ['Between \$1b and \$5b',b4], ['Between \$500m and \$1b',b5], ['Under \$500m',b6]]

df_assets = pd.DataFrame(list_of_lists, columns=['Range of Assets', 'Number of Closures'])
df_assets.style.hide_index()
```




<style  type="text/css" >
</style><table id="T_46540cac_9c74_11ea_8db1_acbc32cefccf" ><thead>    <tr>        <th class="col_heading level0 col0" >Range of Assets</th>        <th class="col_heading level0 col1" >Number of Closures</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow0_col0" class="data row0 col0" >More than $50b</td>
                        <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow0_col1" class="data row0 col1" >1</td>
            </tr>
            <tr>
                                <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow1_col0" class="data row1 col0" >Between \$10b and \$50b</td>
                        <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow1_col1" class="data row1 col1" >8</td>
            </tr>
            <tr>
                                <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow2_col0" class="data row2 col0" >Between \$5b and \$10b</td>
                        <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow2_col1" class="data row2 col1" >6</td>
            </tr>
            <tr>
                                <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow3_col0" class="data row3 col0" >Between \$1b and \$5b</td>
                        <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow3_col1" class="data row3 col1" >58</td>
            </tr>
            <tr>
                                <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow4_col0" class="data row4 col0" >Between \$500m and \$1b</td>
                        <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow4_col1" class="data row4 col1" >58</td>
            </tr>
            <tr>
                                <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow5_col0" class="data row5 col0" >Under \$500m</td>
                        <td id="T_46540cac_9c74_11ea_8db1_acbc32cefccfrow5_col1" class="data row5 col1" >425</td>
            </tr>
    </tbody></table>



Let's visualize this data:


```python
# Make lists out of each column

ranges = []
for row in df_assets['Range of Assets']:
    ranges.append(row)
ranges.reverse()

banknums = []
for row in df_assets['Number of Closures']:
    banknums.append(row)
banknums.reverse()

```


```python
fig, ax = plt.subplots(figsize=(15,12), dpi=300)

ax.bar(ranges, banknums)

ax.set_xlabel('Range of Assets', fontsize=14)
ax.set_ylabel('Number of Closures', fontsize=14)
plt.show()
```


    
![png](bank_closures_in_the_us_files/bank_closures_in_the_us_57_0.png)
    


It becomes more than clear that most of the bank failures were **small, regional institutions**.

<a id=’section_6’></a>

### 6. Acquiring institutions

We'll now look at what institutions -if any- acquired the failed banks. For this, we will set up a table of bank closures, grouped by **acquiring institution**.


```python
cur.execute("""
    DROP TABLE IF EXISTS Acquiring_Institution;
    CREATE TABLE Acquiring_Institution (
        Acquiring_Institution VARCHAR(100) UNIQUE,
        closure_count INTEGER
    );
""")
```


```python
cur.execute("""INSERT INTO Acquiring_Institution (acquiring_institution, closure_count) 
                SELECT bank_list.acqinst, 
                COUNT (*) FROM bank_list GROUP BY bank_list.acqinst;""")
conn.commit()
```


```python
conn.close()
```

Print the 5 institutions with the biggest number of acquisitions.


```python
df = pd.read_sql_table("acquiring_institution", engine)
df.sort_values(by=['closure_count'], inplace=True, ascending=False)
df.head(5).style.hide_index()
```




<style  type="text/css" >
</style><table id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccf" ><thead>    <tr>        <th class="col_heading level0 col0" >acquiring_institution</th>        <th class="col_heading level0 col1" >closure_count</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow0_col0" class="data row0 col0" >No Acquirer</td>
                        <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow0_col1" class="data row0 col1" >31</td>
            </tr>
            <tr>
                                <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow1_col0" class="data row1 col0" >State Bank and Trust Company</td>
                        <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow1_col1" class="data row1 col1" >12</td>
            </tr>
            <tr>
                                <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow2_col0" class="data row2 col0" >First-Citizens Bank & Trust Company</td>
                        <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow2_col1" class="data row2 col1" >11</td>
            </tr>
            <tr>
                                <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow3_col0" class="data row3 col0" >Ameris Bank</td>
                        <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow3_col1" class="data row3 col1" >10</td>
            </tr>
            <tr>
                                <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow4_col0" class="data row4 col0" >U.S. Bank N.A.</td>
                        <td id="T_81bd6c98_9c74_11ea_8db1_acbc32cefccfrow4_col1" class="data row4 col1" >9</td>
            </tr>
    </tbody></table>




```python
plt.figure(figsize=[10,8], dpi=300)
plt.barh(df.acquiring_institution, df.closure_count)

plt.xticks(fontsize=10)
plt.yticks(fontsize=10)
plt.ylim(1,20)
plt.xlim(0,15)
plt.gca().invert_yaxis()

plt.show()
```


    
![png](bank_closures_in_the_us_files/bank_closures_in_the_us_67_0.png)
    


Interestingly, 4 out of the top 5 acquiring institutions are based and operate mainly in the Southeast.

The bank that acquired the most dissolved institutions, **State Bank and Trust Company** was recently acquired by **Cadence Bancorporation**, effective January 1, 2019. 
>"*Funded by private investment capital, State Bank and Trust was opened in 2005 to acquire failed banks in FDIC assisted transactions.*"  
&emsp;[Problem Bank List](http://problembanklist.com/piedmont-community-bank-of-georgia-fails-taken-over-by-state-bank-and-trust-co-0414/), October 14, 2011  

<a id=’section_7’></a>

### 7. Additional Resources

1. [Macro perspective on the global financial crisis: "The Giant Pool of Money" - This American Life](https://www.thisamericanlife.org/355/the-giant-pool-of-money)
2. ["The regional banks: The evolution of the financial sector, Part II" - Brookings](https://www.brookings.edu/research/the-regional-banks-the-evolution-of-the-financial-sector-part-ii/)
3. ["JPMorgan Buys WaMu Deposits; Regulators Seize Thrift" - Bloomberg](https://web.archive.org/web/20121022191024/http://www.bloomberg.com/apps/news?pid=newsarchive&sid=aWxliUXHsOoA&refer=home)
