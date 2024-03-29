---
title: 'FiveThirtyEight Riddler Challenge: ''Transitive Champions'''
author: Josh Fangmeier
date: '2019-04-14'
slug: []
aliases:
  - /post/fivethirtyeight-riddler-challenge-transitive-champions/
tags:
  - 538 riddler
  - basketball
  - mbb
  - wbb
toc: no
images: ~
---



## The Challenge

This weekend, I attempted to solve my first FiveThirtyEight Riddler challenge. Many of these challenges require a bit more probability theory than I'm comfortable with, but [this week's classic challenge](https://fivethirtyeight.com/features/how-many-times-a-day-is-a-broken-clock-right/) hit a subject that I care too much about: college basketball national champions and the bragging rights that come from beating the champ:

> On Sunday, the Baylor Lady Bears won the 2019 NCAA women’s basketball championship, and on Monday, the Virginia Cavaliers did the same on the men’s side.
But what about all of the unsung transitive champions? For example, earlier in the season, Florida State beat Virginia, thereby laying claim to a transitive championship for the Seminoles. And Boston College beat Florida State, claiming one for the Eagles. And IUPUI beat Boston College, and Ball State beat IUPUI, and so on and so on.
Baylor, meanwhile, only lost once, to Stanford, who lost to five teams, and so on.
How many transitive national champions were there this season in the women’s and men’s games? Or, maybe more descriptively, how many teams weren’t transitive national champions? You should include tournament losses in your calculations.

The author (Oliver Roeder) then supplies links to the results of women's and men's basketball for the 2018-2019 season from [masseyratings.com](https://www.masseyratings.com). On first inspection of the game results, they seem to be in a text format that could be scrapped. In addition, I noticed that the [women's link](https://www.masseyratings.com/scores.php?s=305973&sub=305973&all=1) includes 27,266 games, while the [men's link](https://www.masseyratings.com/scores.php?s=305972&sub=11590&all=1) contains only 6,048. The women's results page includes several junior colleges and even my alma mater [Hastings College](https://twitter.com/hcbroncowbb?lang=en), who I know compete at the NAIA level, not NCAA Division I. The men's results page includes only Division I teams, and since the challenge only mentions Baylor and Virginia, I'm assuming we want to compare Division I transitive champions. I used [another link](https://www.masseyratings.com/scores.php?s=305973&sub=11590&all=1) to pull the women's results for 5,638 Division I games.


## Scraping the game results data

I decided to tackle this challenge using the `tidyverse` family of R packages that can scrap the data and wrangle it into a tidy format for further analysis.

One major challenge is that I wrote a function to parse the college names and scores from the Massey Rating text. This involves some very gnarly regular expression writing as you can see.


```r
library(tidyverse)
library(lubridate)
library(rvest)
```


```r
bb_games_process <- function(x){
  read_html(x) %>% 
    html_node(xpath = '/html/body/pre/text()[1]') %>% 
    html_text() %>% 
    enframe() %>% 
    separate_rows(value, sep = "\n") %>% 
    mutate(value = str_squish(value),
           game = substring(value, 12),
           date = ymd(substr(value,1,10)),
           win_team = str_extract(game, "^[\\@]{0,1}[:alpha:]{1,}[:blank:]{0,1}[:punct:]{0,1}[:blank:]{0,1}[:alpha:]{0,1}[:blank:]{0,1}[:alpha:]{1,}[:punct:]{0,1}[:alpha:]{0,1}[:blank:]{0,1}[:alpha:]{1,}[:punct:]{0,1}[:alpha:]{0,1}"),
           win_pts = as.integer(str_extract(game, "(?<=[:alpha:]{1,100}[:blank:]{1})[:digit:]{1,3}")),
           lose_team = str_extract(game, "(?<=[:digit:]{1,3}[:blank:]{1})[\\@]{0,1}[:alpha:]{1,}[:blank:]{0,1}[:punct:]{0,1}[:blank:]{0,1}[:alpha:]{0,1}[:blank:]{0,1}[:alpha:]{1,}[:punct:]{0,1}[:alpha:]{0,1}[:blank:]{0,1}[:alpha:]{1,}[:punct:]{0,1}[:alpha:]{0,1}"),
           lose_pts = as.integer(str_extract(game, "(?<=[:digit:]{1,3}[:blank:]{1}[\\@]{0,1}[:alpha:]{1,100}[:blank:]{0,1}[:punct:]{0,1}[:blank:]{0,1}[:alpha:]{0,1}[:blank:]{0,1}[:alpha:]{1,100}[:punct:]{0,1}[:alpha:]{0,1}[:blank:]{0,1}[:alpha:]{1,100}[:punct:]{0,1}[:alpha:]{0,1}[:blank:]{1})[:digit:]{1,3}"))) %>% 
    select(-name, -value) %>% 
    filter(!is.na(date)) %>% 
    mutate(home_team = if_else(str_detect(win_team, "\\@"), win_team, 
                               if_else(str_detect(lose_team, "\\@"), lose_team, "Neutral Court"))) %>% 
    mutate_at(vars(contains('team')), list(~str_remove(., "\\@"))) %>% 
    cbind(x)
}

#wbb_url <- "https://www.masseyratings.com/scores.php?s=305973&sub=305973&all=1" #Original URL (All WBB games)
wbb_url <- "https://www.masseyratings.com/scores.php?s=305973&sub=11590&all=1" #Corrected URL (only D1 WBB games)
mbb_url <- "https://www.masseyratings.com/scores.php?s=cb2019&sub=ncaa-d1&all=1&sch=1" #Original URL (only D1 MBB games)
urls <- c(wbb_url, mbb_url)

bb_games <- map_dfr(urls, bb_games_process) %>% 
  as_tibble() %>% 
  mutate(sport = if_else(x == wbb_url, "WBB", "MBB")) %>% 
  select(-x)

bb_games %>% tail(n=5)
```

```
## # A tibble: 5 x 8
##   game            date       win_team win_pts lose_team lose_pts home_team sport
##   <chr>           <date>     <chr>      <int> <chr>        <int> <chr>     <chr>
## 1 Texas 81 Lipsc~ 2019-04-04 Texas         81 Lipscomb        66 Neutral ~ MBB  
## 2 South Florida ~ 2019-04-05 South F~      77 DePaul          65 DePaul    MBB  
## 3 Virginia 63 Au~ 2019-04-06 Virginia      63 Auburn          62 Neutral ~ MBB  
## 4 Texas Tech 61 ~ 2019-04-06 Texas T~      61 Michigan~       51 Neutral ~ MBB  
## 5 Virginia 85 Te~ 2019-04-08 Virginia      85 Texas Te~       77 Neutral ~ MBB
```



Using this prepped data, we can identify that there were 543 teams that played in women's games and 650 that played in men's games during the last season.

## Calculating the number of transitive champions
To identify each "transitive champion" in each sport, I looked for where the nation champion lost and pulled a vector of the opponent(s) who beat the champion during the season. I then looped (ugh, I know, I know) through multiple rounds to see who defeated those teams, and so on and so forth. With each loop, I also pasted the number of unique transitive champions who were identified in each round into a data frame for further analysis.

#### Setup

```r
rounds <- 25
wbb_n_champ <- "Baylor"
mbb_n_champ <- "Virginia"
```

#### WBB calculations

```r
wbb_t_champs <- bb_games %>% 
  filter(sport == "WBB" & lose_team %in% wbb_n_champ) %>% 
  pull(win_team) %>% 
  unique()

wbb_degree_sep <- wbb_t_champs %>% length

for(x in 1:rounds){
  wbb_t_champ_beaters <- bb_games %>% 
    filter(sport == "WBB" & lose_team %in% wbb_t_champs) %>% 
    pull(win_team) %>% 
    unique()
  wbb_t_champs <- c(wbb_t_champs, wbb_t_champ_beaters)
  wbb_t_champs <- wbb_t_champs[!wbb_t_champs == wbb_n_champ] #Remove the national champion from the transitive champs vector
  wbb_degree_sep <- rbind(wbb_degree_sep, wbb_t_champs %>% unique() %>% length)
}

wbb_transitive_champs <- wbb_degree_sep %>% 
  as.data.frame() %>% 
  cbind(total_wbb_teams) %>%
  rename(transitive_champions = V1,
         total_teams = total_wbb_teams) %>% 
  mutate(degree_of_separation = row_number(),
         transitive_champ_pct = transitive_champions / total_teams,
         sport = "WBB") 
```

#### MBB calculations

```r
mbb_t_champs <- bb_games %>% 
  filter(sport == "MBB" & lose_team %in% mbb_n_champ) %>% 
  pull(win_team) %>% 
  unique()

mbb_degree_sep <- mbb_t_champs %>% length

for(x in 1:rounds){
  mbb_t_champ_beaters <- bb_games %>% 
    filter(sport == "MBB" & lose_team %in% mbb_t_champs) %>% 
    pull(win_team) %>% 
    unique()
  mbb_t_champs <- c(mbb_t_champs, mbb_t_champ_beaters)
  mbb_t_champs <- mbb_t_champs[!mbb_t_champs == mbb_n_champ] #Remove the national champion from the transitive champs vector
  mbb_degree_sep <- rbind(mbb_degree_sep, mbb_t_champs %>% unique() %>% length)
}

mbb_transitive_champs <- mbb_degree_sep %>% 
  as.data.frame() %>% 
  cbind(total_mbb_teams) %>%
  rename(transitive_champions = V1,
         total_teams = total_mbb_teams) %>% 
  mutate(degree_of_separation = row_number(),
         transitive_champ_pct = transitive_champions / total_teams,
         sport = "MBB") 
```

#### Bringing the transitive champion data together

```r
transitive_champs <- bind_rows(
  wbb_transitive_champs,
  mbb_transitive_champs) %>% 
  mutate(sport = as_factor(sport))
```

## Results!
For the 2018-2019, I identified the following number of "transitive champs":

* Women's Basketball: __360 transitive champions__

* Men's Basketball: __358 transitive champions__

Each sport reached the total number of transitive champs within 8 degrees of separation of the national champion. However, transitive champs comprise 66% of total women's Division I basketball teams compared to 55% in the men's game, as the plot below shows. This could be due to an effect of major conference teams playing each other more in men's basketball (limiting opportunities for minor conference teams to grab a transitive championship), but that hypothesis would have to be tested in further analysis.


```r
transitive_champs %>% 
  ggplot(aes(x=degree_of_separation, y=transitive_champ_pct, color = sport)) +
  labs(title = "How many basketball teams beat a team, who beat a team, \nwho.... beat the national champ?",
       subtitle = "Analysis of 2018-2019 college basketball games") + 
  geom_line() +
  geom_point() +
  theme_minimal() +
  facet_grid(rows = vars(sport), scales = "free") +
  scale_x_continuous("Degrees of Separation from Actual National Champion") +
  scale_y_continuous("Cumulative % of Teams", labels = scales::percent) +
  theme(legend.position = "none")
```

<img src="{{< blogdown/postref >}}index_files/figure-html/t_champs plot-1.png" width="672" />
