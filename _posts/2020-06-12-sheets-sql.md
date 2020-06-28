---
layout: post
title:  "Databases and spreadsheets with SQL"
date:   2020-06-12 15:00:00
categories: [google,sheets,sql]
comments: true
excerpt: "Google Sheets can also be used for data analysis with the amazing and very simple function Query, even to create some interactive charts."
---
There are many ways of achieving similar results when it comes do analyzing and working with data. I am a big fan of flexibility and taking advantage of what each method has to offer.

Some options will be better for working with larger databases while other will do a good job with small ones. Some might be suited for those who are not so well versed with a programming language like Python and others will offer a wider range of resources for dealing with any situation.

As someone with quite some experience on working with spreadsheets, naturally they become an easy go-to way to start analyzing data. However, the two-dimensional way of representing information on the screen will likely become a barrier, even with all lookups and pivot tables in the world. That is why learning a proper database language like SQL is very handy.

While SQL can get quite complex when tables and relationships become numerous, it is also useful even for simple tasks, specially because its syntax is not so scary, being even similar to plain english.

This will be the first post of a series where I will discuss some ways of integrating SQL and SQL-like commands to databases and spreadsheets.

## Google Sheets and the QUERY function

Google Sheets is great. It saves automatically on the cloud, so it can be accessed anywhere and is amazing to share and colaborate. There is a perfect integration with Forms and an interesting integration with other apps like Docs, Slides and Sites to create nice presentations.

I particularly use Sheets more often than Excel. I have a Chrome Browser dedicated to open .xls files, with a plugin to open them directly with Google Drive apps. I like the clean interface and I am used to the functions, syntax and charts. This came from necessity of sharing the spreadsheets and working on them real time, and now I find it great.

It has some limitations, specially when data starts to get bigger. I have tried to work with hundreds of thousands of lines and it was not good, to say the least. But that is not really the goal of this tool. This is for a day-to-day simple use, with some sophisticated tools to make it more interesting.

I will work with a dataset downloaded from Kaggle, uploaded by the newspaper The Guardian, that contains information about extinct and endangered languages throughout the world. The link for the dataset is [here](https://www.kaggle.com/the-guardian/extinct-languages).

## Importing and getting data ready

With Google Sheets, importing .csv data is very easy. Just File > Import and then select the file. Depending on what you select it will open a new spreadsheet, file, or replace the current open one. The initial data looks like this then:

<figure>
  <img src="{{site.url}}/img/gsheets/image.png" alt="Fig 1"/>
  <figcaption><em>Fig 1 - Imported data</em></figcaption>
</figure>

If the original file was in a good format we can continue right away, but if we need to clean up a little bit, we can also do it. A good way to start is to apply filters to all the columns to make it possible to select some values and see if it works fine. A nice trick can be done on columns that are expected to receive numbers: select the entire column of interest and check if you can see the sum of the values. If you can't, probably there are some strings in the middle. The solution is to just select the column, click Format > Number > Automatic, or any other format needed.

<figure>
  <img src="{{site.url}}/img/gsheets/image-2.png" alt="Fig 2" width="40%" height="40%"/>
  <figcaption><em>Fig 2 - Calculating the sum of a column tells us if all values are numeric</em></figcaption>
</figure>

When we decide to start working with our queries, a good practice is to create a new spreadsheet inside the file.

Because the function QUERY will not use the column headers, but the letters of the spreadsheet, it can be useful to have a small cheat-code with the column names so we can know what we are selecting. This is definitely not mandatory though.

In the data sheet we can select all the columns we have information on and then click Data > Named ranges. We will see a box like this to create a name for this range to be used in future references:

<figure>
  <img src="{{site.url}}/img/gsheets/image-3.png" alt="Fig 3" width="40%" height="40%"/>
  <figcaption><em>Fig 3 - Creating data range</em></figcaption>
</figure>

## Creating a Query

Now, on the new spreadsheet we will use on the A1 cell the function QUERY. The function has three parameters: which table will be searched; the SQL-like command; and if the table has or not headers.

When we start typing the table name we see this:

<figure>
  <img src="{{site.url}}/img/gsheets/image-4.png" alt="Fig 4" width="40%" height="40%"/>
  <figcaption><em>Fig 4 - Range referenced from 'dataset'</em></figcaption>
</figure>

An autocomplete feature is good when we have more named ranges, so we can select the correct one while writing the function.

A big difference from SQL is that here we specify the table for the function and then the select command afterwards is related to that table, so we don't use the traditional select * from table. If we want to show the entire data, we type:

```
=QUERY(dataset;"select *")
```

<figure>
  <img src="{{site.url}}/img/gsheets/image-5.png" alt="Fig 5"/>
  <figcaption><em>Fig 5 - Selecting all data</em></figcaption>
</figure>

Using the function on the A1 cell will make the query show its results on all the next columns and rows that are empty. It is important then to have "clear space" on the spreadsheet. Selecting any cell other than the A1 will show the data as a value, not as a formula.

We can now make filters, ranks and aggregations for example.

```
=QUERY(dataset;"select B, E, H, I, K, M, N, O where E = 'Brazil' and K < 10000")
```

<figure>
  <img src="{{site.url}}/img/gsheets/image-6.png" alt="Fig 6"/>
  <figcaption><em>Fig 6 - Selecting some columns where the country field is 'Brazil' and there are less than 10.000 speakers for that language</em></figcaption>
</figure>

```
=QUERY(dataset;"select count(B), E, sum(K)/count(B) where K < 5000 group by E")
```

<figure>
  <img src="{{site.url}}/img/gsheets/image-7.png" alt="Fig 7" width="60%" height="60%"/>
  <figcaption><em>Fig 7 - Selecting the count of languages with less than 5.000 speakers for each group of values in the Countries column, the value of the Countries column and the ratio between the number of people speaking those languages and the count of the languages, and finally a label for this calculation</em></figcaption>
</figure>

```
=QUERY(dataset;"select B, E, H, K, where E = 'Italy' order by K desc limit 5")
```

<figure>
  <img src="{{site.url}}/img/gsheets/image-8.png" alt="Fig 8" width="60%" height="60%"/>
  <figcaption><em>Fig 8 - Selecting languages spoken in Italy, sorting by highest to lowest number of speakers and showing only the first 5 results</em></figcaption>
</figure>

This already is a very powerful way to filter data. Google Query Language [documentation](https://developers.google.com/chart/interactive/docs/querylanguage) has the entire list of commands to be used.

We can make things easier for users that are not so experienced with spreadsheets and create an interactive query tool as well. While this will require some work in order to be sure to give plenty of flexibility to the user for the data search, a simple demonstration could be like this:

```
=QUERY(dataset;"select B, E, H, K where E contains '"&B2&"' and K <= "&B3)
```

<figure>
  <img src="{{site.url}}/img/gsheets/image-9.png" alt="Fig 9" width="70%" height="70%"/>
  <figcaption><em>Fig 9 - Here we created a little search box with a field for the countries and a space to enter the max number of speakers. As soon as these values are inserted the query selects the data accordingly</em></figcaption>
</figure>

The dropdown menu can be done with a right click on the cell and then selecting 'Data Validation'.

<figure>
  <img src="{{site.url}}/img/gsheets/image-10.png" alt="Fig 10" width="40%" height="40%"/>
  <figcaption><em>Fig 10 - Creating input validation</em></figcaption>
</figure>

We can also have some automated charts. In this example I show a simple example where we plot a bar chart with the top-10 languages in terms of number of speakers (when they have equal or less than 1 million people speaking them).

<figure>
  <img src="{{site.url}}/img/gsheets/giflang-3.gif" alt="Fig 11"/>
  <figcaption><em>Fig 11 - Automated chart based on search criteria</em></figcaption>
</figure>

The title of the chart is written on a normal cell, with some concatenation to get the name of what was written in the Country/Countries field.

Another interesting tip is that the query can perfectly be hidden in another spreadsheet (or maybe just put the chart on top of the query result) and the final user can have access only to the search box and the chart.

It is important that the chart should not be limited by its own parameters; that means that the user shouldn't put the number of rows to be plotted. Rather than that, the best idea is to just select the entire columns (or, in this case, A6:D) and use the limit command to specify how many observations will be selected.

## Conclusion

Google Sheets is a great free tool that allows some very nice creations using not so big datasets. There are some issues, mainly the lack of a join command to join and merge different tables. It can still be done, however it is not optimal. We will explore these options on future posts.

For simple work and for dealing with online Google Forms, this kind of approach using QUERY, automated charts and other kind of reports is amazing and produces good and quick results.