# Анализ данных сетевого трафика с использованием аналитической
in-memory СУБД DuckDB


## Цель

1.  Изучить возможности СУБД [DuckDB](https://duckdb.org//) для
    обработки и анализ больших данных
2.  Получить навыки применения DuckDB совместно с языком
    программирования R
3.  Получить навыки анализа `метаинфомации о сетевом трафике`
4.  Получить навыки применения облачных технологий хранения, подготовки
    и анализа данных: [Yandex Object
    Storage](https://yandex.cloud/ru/docs/storage/), [Rstudio
    Server](https://posit.co/products/open-source/rstudio-server/)

## ️Исходные данные

1.  R 4.4.1
2.  RStudio 2024.04.2+764
3.  Yandex Cloud

## ️Общий план выполнения

Используя язык программирования `R`, библиотеку `arrow` и облачную
`IDE Rstudio Server`, развернутую в `Yandex Cloud`, выполнить задания и
составить отчет.

## Содержание ЛР

### Подготовка данных

1.  **Загрузка пакетов**

``` r
library(DBI)
```

    Warning: пакет 'DBI' был собран под R версии 4.4.2

``` r
library(duckdb)
```

    Warning: пакет 'duckdb' был собран под R версии 4.4.2

``` r
library(dplyr)
```


    Присоединяю пакет: 'dplyr'

    Следующие объекты скрыты от 'package:stats':

        filter, lag

    Следующие объекты скрыты от 'package:base':

        intersect, setdiff, setequal, union

1.  **Установка соединения**

``` r
con <- dbConnect(duckdb())
```

``` r
dbGetQuery(con, "CREATE TABLE new_tbl AS SELECT * FROM read_parquet('./../Lab 7/data/tm_data.pqt');")
```

    Warning in dbFetch(rs, n = n, ...): Should not call dbFetch() on results that
    do not come from SELECT, got CREATE

    таблица данных с 0 колонок и 0 строками

### Задание 1: Найдите утечку данных из Вашей сети

``` r
dbGetQuery(con, "
           SELECT src FROM new_tbl
           WHERE (src LIKE '12.%' or src LIKE '13.%' or src LIKE '14.%') and 
           NOT (dst LIKE '12.%' or dst LIKE '13.%' or dst LIKE '14.%')
           GROUP BY src
           ORDER BY sum(bytes) DESC
           LIMIT 1
           ") %>% 
  knitr::kable()
```

<table>
<thead>
<tr class="header">
<th style="text-align: left;">src</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;">13.37.84.125</td>
</tr>
</tbody>
</table>

### Задание 2: Найдите утечку данных 2

``` r
dbGetQuery(con, "
           SELECT extract(hour FROM epoch_ms(cast(timestamp as bigint))) as hour, STDDEV(bytes) as sd FROM new_tbl
           WHERE (src LIKE '12.%' or src LIKE '13.%' or src LIKE '14.%') and 
           NOT (dst LIKE '12.%' or dst LIKE '13.%' or dst LIKE '14.%')
           GROUP BY hour
           ORDER BY sd DESC
           LIMIT 1
           ") %>%
  knitr::kable()
```

<table>
<thead>
<tr class="header">
<th style="text-align: right;">hour</th>
<th style="text-align: right;">sd</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">7</td>
<td style="text-align: right;">40316.72</td>
</tr>
</tbody>
</table>

``` r
dbGetQuery(con, "
           SELECT src FROM new_tbl
           WHERE (src LIKE '12.%' or src LIKE '13.%' or src LIKE '14.%') and 
           NOT (dst LIKE '12.%' or dst LIKE '13.%' or dst LIKE '14.%') and
           extract(hour FROM epoch_ms(cast(timestamp as bigint))) == 7
           GROUP BY src
           ORDER BY sum(bytes) DESC
           LIMIT 1
           ") %>%
  knitr::kable()
```

<table>
<thead>
<tr class="header">
<th style="text-align: left;">src</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;">12.55.77.96</td>
</tr>
</tbody>
</table>

### Задание 3: Найдите утечку данных 3

``` r
dbGetQuery(con, "
           SELECT port, count(DISTINCT src) as count_src FROM new_tbl
           WHERE (src LIKE '12.%' or src LIKE '13.%' or src LIKE '14.%') and 
           NOT (dst LIKE '12.%' or dst LIKE '13.%' or dst LIKE '14.%')
           GROUP BY port
           ORDER BY count_src
           ") %>%
  knitr::kable()
```

<table>
<thead>
<tr class="header">
<th style="text-align: right;">port</th>
<th style="text-align: right;">count_src</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">21</td>
<td style="text-align: right;">1</td>
</tr>
<tr class="even">
<td style="text-align: right;">32</td>
<td style="text-align: right;">1</td>
</tr>
<tr class="odd">
<td style="text-align: right;">36</td>
<td style="text-align: right;">1</td>
</tr>
<tr class="even">
<td style="text-align: right;">31</td>
<td style="text-align: right;">1</td>
</tr>
<tr class="odd">
<td style="text-align: right;">95</td>
<td style="text-align: right;">1</td>
</tr>
<tr class="even">
<td style="text-align: right;">78</td>
<td style="text-align: right;">1</td>
</tr>
<tr class="odd">
<td style="text-align: right;">51</td>
<td style="text-align: right;">24</td>
</tr>
<tr class="even">
<td style="text-align: right;">96</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">75</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">56</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">65</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">80</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">29</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">40</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">34</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">82</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">112</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">105</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">123</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">121</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">22</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">42</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">26</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">106</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">117</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">119</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">115</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">94</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">27</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">89</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">57</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">79</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">37</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">52</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">92</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">68</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">50</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">39</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">81</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">114</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">55</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">90</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">61</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">72</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">44</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">77</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">74</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">25</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">23</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">124</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">102</td>
<td style="text-align: right;">1000</td>
</tr>
<tr class="even">
<td style="text-align: right;">118</td>
<td style="text-align: right;">1000</td>
</tr>
</tbody>
</table>

``` r
dbGetQuery(con, "
           WITH ranked_src AS (
           SELECT port, src, SUM(bytes) AS total_bytes,
           ROW_NUMBER() OVER (PARTITION BY port ORDER BY SUM(bytes) DESC) AS rn
           FROM new_tbl
           GROUP BY port, src
           )

           SELECT port, src, total_bytes FROM ranked_src
           WHERE rn <= 2
           ") %>%
  knitr::kable()
```

<table>
<thead>
<tr class="header">
<th style="text-align: right;">port</th>
<th style="text-align: left;">src</th>
<th style="text-align: right;">total_bytes</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">94</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2643483</td>
</tr>
<tr class="even">
<td style="text-align: right;">94</td>
<td style="text-align: left;">13.39.46.94</td>
<td style="text-align: right;">2544630</td>
</tr>
<tr class="odd">
<td style="text-align: right;">31</td>
<td style="text-align: left;">12.55.77.96</td>
<td style="text-align: right;">233345180</td>
</tr>
<tr class="even">
<td style="text-align: right;">32</td>
<td style="text-align: left;">13.37.84.125</td>
<td style="text-align: right;">1989408807</td>
</tr>
<tr class="odd">
<td style="text-align: right;">52</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">95544391</td>
</tr>
<tr class="even">
<td style="text-align: right;">52</td>
<td style="text-align: left;">12.45.94.34</td>
<td style="text-align: right;">90887660</td>
</tr>
<tr class="odd">
<td style="text-align: right;">39</td>
<td style="text-align: left;">12.54.40.45</td>
<td style="text-align: right;">92803040</td>
</tr>
<tr class="even">
<td style="text-align: right;">39</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">92732957</td>
</tr>
<tr class="odd">
<td style="text-align: right;">78</td>
<td style="text-align: left;">13.37.84.125</td>
<td style="text-align: right;">2018366254</td>
</tr>
<tr class="even">
<td style="text-align: right;">106</td>
<td style="text-align: left;">13.42.70.40</td>
<td style="text-align: right;">387600</td>
</tr>
<tr class="odd">
<td style="text-align: right;">106</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">367710</td>
</tr>
<tr class="even">
<td style="text-align: right;">40</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">92658489</td>
</tr>
<tr class="odd">
<td style="text-align: right;">40</td>
<td style="text-align: left;">14.57.60.122</td>
<td style="text-align: right;">92343272</td>
</tr>
<tr class="even">
<td style="text-align: right;">77</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2665587</td>
</tr>
<tr class="odd">
<td style="text-align: right;">77</td>
<td style="text-align: left;">13.48.72.30</td>
<td style="text-align: right;">2609139</td>
</tr>
<tr class="even">
<td style="text-align: right;">42</td>
<td style="text-align: left;">13.42.70.40</td>
<td style="text-align: right;">2036251</td>
</tr>
<tr class="odd">
<td style="text-align: right;">42</td>
<td style="text-align: left;">14.51.30.86</td>
<td style="text-align: right;">1866452</td>
</tr>
<tr class="even">
<td style="text-align: right;">115</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">95397988</td>
</tr>
<tr class="odd">
<td style="text-align: right;">115</td>
<td style="text-align: left;">12.56.32.111</td>
<td style="text-align: right;">94594841</td>
</tr>
<tr class="even">
<td style="text-align: right;">22</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2651111</td>
</tr>
<tr class="odd">
<td style="text-align: right;">22</td>
<td style="text-align: left;">12.56.32.111</td>
<td style="text-align: right;">2607362</td>
</tr>
<tr class="even">
<td style="text-align: right;">29</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">96749594</td>
</tr>
<tr class="odd">
<td style="text-align: right;">29</td>
<td style="text-align: left;">12.46.21.45</td>
<td style="text-align: right;">94125106</td>
</tr>
<tr class="even">
<td style="text-align: right;">114</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">95605415</td>
</tr>
<tr class="odd">
<td style="text-align: right;">114</td>
<td style="text-align: left;">14.57.60.122</td>
<td style="text-align: right;">91184443</td>
</tr>
<tr class="even">
<td style="text-align: right;">90</td>
<td style="text-align: left;">13.42.70.40</td>
<td style="text-align: right;">63630</td>
</tr>
<tr class="odd">
<td style="text-align: right;">90</td>
<td style="text-align: left;">14.57.60.122</td>
<td style="text-align: right;">58254</td>
</tr>
<tr class="even">
<td style="text-align: right;">51</td>
<td style="text-align: left;">12.48.48.60</td>
<td style="text-align: right;">8022</td>
</tr>
<tr class="odd">
<td style="text-align: right;">51</td>
<td style="text-align: left;">13.32.122.43</td>
<td style="text-align: right;">8022</td>
</tr>
<tr class="even">
<td style="text-align: right;">56</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">93754710</td>
</tr>
<tr class="odd">
<td style="text-align: right;">56</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">91830199</td>
</tr>
<tr class="even">
<td style="text-align: right;">113</td>
<td style="text-align: left;">15.104.76.58</td>
<td style="text-align: right;">163171</td>
</tr>
<tr class="odd">
<td style="text-align: right;">82</td>
<td style="text-align: left;">13.39.46.94</td>
<td style="text-align: right;">2610547</td>
</tr>
<tr class="even">
<td style="text-align: right;">82</td>
<td style="text-align: left;">12.45.94.34</td>
<td style="text-align: right;">2575561</td>
</tr>
<tr class="odd">
<td style="text-align: right;">61</td>
<td style="text-align: left;">13.48.72.30</td>
<td style="text-align: right;">2613819</td>
</tr>
<tr class="even">
<td style="text-align: right;">61</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2609141</td>
</tr>
<tr class="odd">
<td style="text-align: right;">92</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">93301619</td>
</tr>
<tr class="even">
<td style="text-align: right;">92</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">92163250</td>
</tr>
<tr class="odd">
<td style="text-align: right;">34</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2665327</td>
</tr>
<tr class="even">
<td style="text-align: right;">34</td>
<td style="text-align: left;">14.51.30.86</td>
<td style="text-align: right;">2606751</td>
</tr>
<tr class="odd">
<td style="text-align: right;">83</td>
<td style="text-align: left;">13.39.46.94</td>
<td style="text-align: right;">9077865</td>
</tr>
<tr class="even">
<td style="text-align: right;">83</td>
<td style="text-align: left;">13.53.114.85</td>
<td style="text-align: right;">8998580</td>
</tr>
<tr class="odd">
<td style="text-align: right;">118</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">96395109</td>
</tr>
<tr class="even">
<td style="text-align: right;">118</td>
<td style="text-align: left;">12.45.94.34</td>
<td style="text-align: right;">91022238</td>
</tr>
<tr class="odd">
<td style="text-align: right;">65</td>
<td style="text-align: left;">12.46.21.45</td>
<td style="text-align: right;">94762319</td>
</tr>
<tr class="even">
<td style="text-align: right;">65</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">92713624</td>
</tr>
<tr class="odd">
<td style="text-align: right;">50</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2617798</td>
</tr>
<tr class="even">
<td style="text-align: right;">50</td>
<td style="text-align: left;">13.39.46.94</td>
<td style="text-align: right;">2600654</td>
</tr>
<tr class="odd">
<td style="text-align: right;">102</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">96751789</td>
</tr>
<tr class="even">
<td style="text-align: right;">102</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">92565969</td>
</tr>
<tr class="odd">
<td style="text-align: right;">81</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">94181529</td>
</tr>
<tr class="even">
<td style="text-align: right;">81</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">92997519</td>
</tr>
<tr class="odd">
<td style="text-align: right;">74</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">97209193</td>
</tr>
<tr class="even">
<td style="text-align: right;">74</td>
<td style="text-align: left;">14.57.60.122</td>
<td style="text-align: right;">92883538</td>
</tr>
<tr class="odd">
<td style="text-align: right;">26</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2648846</td>
</tr>
<tr class="even">
<td style="text-align: right;">26</td>
<td style="text-align: left;">13.48.72.30</td>
<td style="text-align: right;">2634252</td>
</tr>
<tr class="odd">
<td style="text-align: right;">44</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">92441060</td>
</tr>
<tr class="even">
<td style="text-align: right;">44</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">91953507</td>
</tr>
<tr class="odd">
<td style="text-align: right;">112</td>
<td style="text-align: left;">13.42.70.40</td>
<td style="text-align: right;">61614</td>
</tr>
<tr class="even">
<td style="text-align: right;">112</td>
<td style="text-align: left;">12.45.94.34</td>
<td style="text-align: right;">57582</td>
</tr>
<tr class="odd">
<td style="text-align: right;">121</td>
<td style="text-align: left;">12.45.94.34</td>
<td style="text-align: right;">2627167</td>
</tr>
<tr class="even">
<td style="text-align: right;">121</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">2602063</td>
</tr>
<tr class="odd">
<td style="text-align: right;">119</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">97987126</td>
</tr>
<tr class="even">
<td style="text-align: right;">119</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">92939771</td>
</tr>
<tr class="odd">
<td style="text-align: right;">79</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2651337</td>
</tr>
<tr class="even">
<td style="text-align: right;">79</td>
<td style="text-align: left;">13.48.72.30</td>
<td style="text-align: right;">2567508</td>
</tr>
<tr class="odd">
<td style="text-align: right;">23</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2642312</td>
</tr>
<tr class="even">
<td style="text-align: right;">23</td>
<td style="text-align: left;">12.56.32.111</td>
<td style="text-align: right;">2618173</td>
</tr>
<tr class="odd">
<td style="text-align: right;">89</td>
<td style="text-align: left;">13.48.72.30</td>
<td style="text-align: right;">93897598</td>
</tr>
<tr class="even">
<td style="text-align: right;">89</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">93469444</td>
</tr>
<tr class="odd">
<td style="text-align: right;">27</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2681815</td>
</tr>
<tr class="even">
<td style="text-align: right;">27</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">2597603</td>
</tr>
<tr class="odd">
<td style="text-align: right;">57</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">93583319</td>
</tr>
<tr class="even">
<td style="text-align: right;">57</td>
<td style="text-align: left;">12.45.94.34</td>
<td style="text-align: right;">93285514</td>
</tr>
<tr class="odd">
<td style="text-align: right;">117</td>
<td style="text-align: left;">13.42.70.40</td>
<td style="text-align: right;">2232000</td>
</tr>
<tr class="even">
<td style="text-align: right;">117</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2122500</td>
</tr>
<tr class="odd">
<td style="text-align: right;">124</td>
<td style="text-align: left;">12.30.96.87</td>
<td style="text-align: right;">368891</td>
</tr>
<tr class="even">
<td style="text-align: right;">124</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">58506</td>
</tr>
<tr class="odd">
<td style="text-align: right;">21</td>
<td style="text-align: left;">13.37.84.125</td>
<td style="text-align: right;">2027501066</td>
</tr>
<tr class="even">
<td style="text-align: right;">25</td>
<td style="text-align: left;">13.42.70.40</td>
<td style="text-align: right;">62832</td>
</tr>
<tr class="odd">
<td style="text-align: right;">25</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">59010</td>
</tr>
<tr class="even">
<td style="text-align: right;">55</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">93692657</td>
</tr>
<tr class="odd">
<td style="text-align: right;">55</td>
<td style="text-align: left;">14.57.60.122</td>
<td style="text-align: right;">90550878</td>
</tr>
<tr class="even">
<td style="text-align: right;">80</td>
<td style="text-align: left;">12.56.32.111</td>
<td style="text-align: right;">2645993</td>
</tr>
<tr class="odd">
<td style="text-align: right;">80</td>
<td style="text-align: left;">14.51.30.86</td>
<td style="text-align: right;">2585943</td>
</tr>
<tr class="even">
<td style="text-align: right;">37</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">94819664</td>
</tr>
<tr class="odd">
<td style="text-align: right;">37</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">91064449</td>
</tr>
<tr class="even">
<td style="text-align: right;">105</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">93963280</td>
</tr>
<tr class="odd">
<td style="text-align: right;">105</td>
<td style="text-align: left;">14.51.75.107</td>
<td style="text-align: right;">91519961</td>
</tr>
<tr class="even">
<td style="text-align: right;">36</td>
<td style="text-align: left;">13.37.84.125</td>
<td style="text-align: right;">2070876332</td>
</tr>
<tr class="odd">
<td style="text-align: right;">68</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2722595</td>
</tr>
<tr class="even">
<td style="text-align: right;">68</td>
<td style="text-align: left;">14.51.30.86</td>
<td style="text-align: right;">2579395</td>
</tr>
<tr class="odd">
<td style="text-align: right;">96</td>
<td style="text-align: left;">14.57.50.29</td>
<td style="text-align: right;">2625035</td>
</tr>
<tr class="even">
<td style="text-align: right;">96</td>
<td style="text-align: left;">12.45.94.34</td>
<td style="text-align: right;">2608000</td>
</tr>
<tr class="odd">
<td style="text-align: right;">75</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">93566770</td>
</tr>
<tr class="even">
<td style="text-align: right;">75</td>
<td style="text-align: left;">12.46.21.45</td>
<td style="text-align: right;">92131744</td>
</tr>
<tr class="odd">
<td style="text-align: right;">72</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2636769</td>
</tr>
<tr class="even">
<td style="text-align: right;">72</td>
<td style="text-align: left;">14.51.30.86</td>
<td style="text-align: right;">2565667</td>
</tr>
<tr class="odd">
<td style="text-align: right;">95</td>
<td style="text-align: left;">13.37.84.125</td>
<td style="text-align: right;">2031985904</td>
</tr>
<tr class="even">
<td style="text-align: right;">123</td>
<td style="text-align: left;">13.42.70.40</td>
<td style="text-align: right;">2251500</td>
</tr>
<tr class="odd">
<td style="text-align: right;">123</td>
<td style="text-align: left;">12.59.25.34</td>
<td style="text-align: right;">2062500</td>
</tr>
</tbody>
</table>

``` r
dbGetQuery(con, "
           SELECT src FROM new_tbl
           WHERE (src LIKE '12.%' or src LIKE '13.%' or src LIKE '14.%') and 
           NOT (dst LIKE '12.%' or dst LIKE '13.%' or dst LIKE '14.%') and
           port == 124
           GROUP BY src
           ORDER BY sum(bytes) DESC
           LIMIT 1
           ") %>%
  knitr::kable()
```

<table>
<thead>
<tr class="header">
<th style="text-align: left;">src</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;">12.30.96.87</td>
</tr>
</tbody>
</table>

## ️Оценка результата

Был проведён анализ сегментированной корпоративной сети с помощью
аналитической in-memory СУБД `DuckDB`

## ️Вывод

В результате выполнения работы были:

-   изучены возможности СУБД [DuckDB](https://duckdb.org//) для
    обработки и анализ больших данных
-   получены навыки применения DuckDB совместно с языком
    программирования R
-   получены навыки анализа `метаинфомации о сетевом трафике`
