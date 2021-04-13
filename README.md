# FPI-Scrape

## Comparison of ESPN FPI (Football Power Index) to Las Vegas betting odds in order to recommend statistically favorable bets

### Project Goals:
- Web Scraping
- Implied odds calculations
- Data Analysis
- Bet Recommendations


### Web Scraping
My web scraping required the following modules:
```python
import requests
import pandas as pd
from bs4 import BeautifulSoup
```

I used the pandas `read_html()` function to scrape Las Vegas betting odds data from [ESPN chalk](https://www.espn.com/chalk/)  
I defined the following function to get up to date betting data:

```python
def getESPNlines():
    url = 'https://www.espn.com/nfl/lines'
    dfs = pd.read_html(url)
    
    return dfs
```  
    
The ESPN FPI data was in a more complicated HTML format, so I used `BeautifulSoup` to scrape that. Here is a sample of the code:

```python
# up to date FPI function
# returns pandas dataframe with current FPI table

def getFPI():
    url = 'https://www.espn.com/nfl/fpi'
    r = requests.get(url)
    html = r.text
    
    # ESPN splits the FPI table into two sides
    
    # put the left table (team names) into a pandas dataframe    
    soup = BeautifulSoup(html)
    table1 = soup.find('table', {"class": "Table Table--align-right Table--fixed Table--fixed-left"})
    rows = table1.find_all('tr')
    teams_data = []
    for row in rows[2:]:
        cols = row.find_all('td')
        cols = [element.text.strip() for element in cols]
        teams_data.append([element for element in cols if element])   

    teams_df = pd.DataFrame(teams_data)
    
    # put the right side table (FPI and other stats) into a pandas dataframe
    table2 = soup.find('table', {"class": "Table Table--align-right"})
    rows = table2.find_all('tr')
    stats_data = []
    for row in rows[2:]:
        cols = row.find_all('td')
        cols = [element.text.strip() for element in cols]
        stats_data.append([element for element in cols if element])   

    stats_df = pd.DataFrame(stats_data)
    
    # combine into one dataframe and update headings
    df = pd.merge(teams_df, stats_df, left_index=True, right_index=True)
    headers = {'0_x': 'TEAM', 
               '0_y': 'W-L', 
               1 : 'FPI', 
               2: 'RK', 
               3: 'TRND', 
               4: 'OFF', 
               5: 'DEF', 
               6: 'ST', 
               7: 'SOS', 
               8: 'REM_SOS', 
               9: 'AVG_WP'}
    df = df.rename(index=str,columns=headers)
    return df
```

### Implied Odds Calculations
#### Negative American Odds (favorite)
implied probability = negative odds / (negative odds + 100) * 100

####  Positive American Odds (underdog)
implied probability = 100 / (positive odds + 100) * 100  

I calculated the implied winning probability for each NFL team based on the Vegas odds and added it as a column to the dataframe. Then I converted the week's schedule to a dataframe, organized the columns, indexed by team, and concanated it with the implied odds:
```python
lines = pd.concat(lines_list)
lines = lines[['TEAM', 'ML', 'SPREAD', 'TOTAL', 'FPI', 'Implied_Odds', 'ML_Edge', 'U/F?']]
team_lines = lines.set_index('TEAM')

points_FPI = getFPI()
points_FPI = points_FPI.set_index('TEAM')
points_FPI['FPI'] = points_FPI['FPI'].astype(float)
```

### Data Analysis
ESPN FPI gives both a win probability and an expected margin of victory against an average team. An above average team will have a positive value, an average team will have a value of zero, and a below average team will have a negative value. I compared the FPI win probability to the implied Vegas win probability I calculated above. I also compared the expected margin of victory to the FPI predicted margin of victory. I generated a dataframe for each comparison, and the head of each dataframe is shown below:  
##### Money Line Comparison
![](https://github.com/jmfinnegan12/FPI-Scrape/blob/main/Photos/ML_table.PNG)  
##### Point Spread Comparison  
![](https://github.com/jmfinnegan12/FPI-Scrape/blob/main/Photos/Spread_Table.PNG)

### Bet Recommendations
The output dataframes shown above are sorted by the most statistically favorable bets, also known as 'edge', in descending order. Generally the Las Vegas betting markets and ESPN's FPI algorithms are very close, and according to [Wikipedia](https://en.wikipedia.org/wiki/Football_Power_Index#cite_note-2), FPI outperformed Las Vegas closing lines in 2016. This fact presents an interesting opportunity for back testing on historical FPI and betting data. A model could be developed based on historical data to identify where FPI could have an advantage over Vegas and lead to more proffitable betting opportunities. As far as I am aware, historical FPI data is not available, so for now this project will just be a fun way to add a bit more excitement to the NFL season
