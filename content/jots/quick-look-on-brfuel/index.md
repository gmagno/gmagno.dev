---
date: "2019-11-03"
draft: false
showpagemeta: true
title: "A quick look at fuel prices in Brazil"
---

> A simple [dashboard](https://gmagno.github.io/brfuel) showing how fuel prices have evolved in Brazil since 2004 until 2018. Just a set of plots organized by time and brazillian regions. Results discussion is left to the reader.

{{< figure src="/jots/quick-look-on-brfuel/austrian-national-library-aUhBP3s4BxY-unsplash.jpg" caption="Photo by [Austrian National Library](https://unsplash.com/photos/aUhBP3s4BxY)" link="https://unsplash.com/photos/aUhBP3s4BxY" >}}

The dataset behind the plots is provided by the brazillian government through [ANP](http://www.anp.gov.br) (*Agência Nacional do Petróleo, Gás Natural e Biocombustíveis*).

## [Fuel prices in Brazil](https://gmagno.github.io/brfuel)

The dashboard tries to address three questions:

1. How do fuel prices vary, in average, among distributors?
2. How do fuel prices vary, in average, between states?
3. What is the average profit margin by state?

The dataset is modestly sized, with information about 554 municipalities, 252
distributors, 25459 fuel stations, and for a period between 2004 and 2018.

> **ℹ️ Note:**
> Some of the plots regarding NGV fuel may seem incomplete for different states.
> I can only suspect that either these states have not sold NGV in the past, or the dataset provided by ANP is missing data.
> Also, and since DIESEL S10 was only introduced in 2013, the respective plots only show data for the period between 2013 and 2018.

The dashboard was written in [Python](https://python.org), powered by [Pandas](https://pandas.pydata.org) and [Bokeh](https://bokeh.org), and rendered with [Jinja](https://jinja.palletsprojects.com) into a [Bootstrap](https://getbootstrap.com)" styled single HTML file.
