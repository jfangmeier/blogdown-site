---
title: 'FiveThirtyEight Riddler Challenge: Can You Find The Fish In State Names?'
author: Josh Fangmeier
date: '2020-05-25'
slug: []
aliases:
  - /post/riddler-can-you-find-the-fish-in-state-names/
categories: []
tags:
  - 538 riddler
toc: no
images: ~
---



## The Challenge

This weekend, I attempted to solve my second FiveThirtyEight Riddler challenge:

> Ohio is the only state whose name doesn’t share any letters with the word “mackerel.” It’s strange, but it’s true. But that isn’t the only pairing of a state and a word you can say that about — it’s not even the only fish! Kentucky has “goldfish” to itself, Montana has “jellyfish” and Delaware has “monkfish,” just to name a few. What is the longest “mackerel?” That is, what is the longest word that doesn’t share any letters with exactly one state? (If multiple “mackerels” are tied for being the longest, can you find them all?)
Extra credit: Which state has the most “mackerels?” That is, which state has the most words for which it is the only state without any letters in common with those words?

## Data prep

```r
library(tidyverse)
library(httr)
library(rvest)

words_raw <- "https://norvig.com/ngrams/word.list" %>% 
  GET() %>% 
  read_html() %>% 
  html_text()

words <- str_split(words_raw, "\n")[[1]]

states <- state.name %>% 
  str_to_lower(.) %>% 
  str_remove_all(., " ")
```

## Calculating string intersections (letters in common)

```r
state_word_intersect <- 
  tidyr::expand_grid(words, states) %>% 
  mutate(intersect = map2(words, states, ~Reduce(intersect, strsplit(c(.x, .y), split = "")))) %>% 
  mutate(intersect_length = map_dbl(intersect, length))
```

## Solution

```r
word_single_state <- 
  state_word_intersect %>% 
  filter(intersect_length == 0) %>% 
  group_by(words) %>% 
  mutate(state_matches = n()) %>% 
  ungroup() %>% 
  filter(state_matches == 1)

word_single_state %>% 
  mutate(word_length = str_length(words),
         max_length = max(word_length, na.rm = TRUE)) %>% 
  filter(word_length == max_length) %>% 
  select(words, word_length, states)
```

```
## # A tibble: 2 x 3
##   words                   word_length states     
##   <chr>                         <int> <chr>      
## 1 counterproductivenesses          23 alabama    
## 2 hydrochlorofluorocarbon          23 mississippi
```

## Extra credit

```r
word_single_state %>% 
  group_by(states) %>% 
  mutate(word_count = n()) %>% 
  ungroup() %>% 
  mutate(max_word_count = max(word_count)) %>% 
  filter(word_count == max_word_count) %>% 
  distinct(states, word_count)
```

```
## # A tibble: 1 x 2
##   states word_count
##   <chr>       <int>
## 1 ohio        11342
```
