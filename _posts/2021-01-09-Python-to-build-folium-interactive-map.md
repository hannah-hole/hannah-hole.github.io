---
title: >-
    Using Python to Build an Interactive Income Map of the UK
---

## Project Background

For the project in our first semester Python class, we were given total freedom with what we were able to build. I particularly love data visualization, so I decided to look at using Python’s data visualization capabilities to display open source UK income data in a way that was interactive and easy to understand. 
I soon discovered Gross Disposable Household Income data at the UK Local Authority level as the most granular data publicly available and that the Folium library/package would be the best way to approach this project. 
My objective was to produce a map that displayed:
*Markers displaying income information for each Local Authority
*Bottom 10% and top 10% of Local Authorities by income 
*Choropleth and boundary overlays
*A Local Authority search tool

[Folium documentation](https://python-visualization.github.io/folium/modules.html) and [Folium Plugins documentation](https://python-visualization.github.io/folium/plugins.html) is your best friend here. There are so many more features available in Folium than are shown here! Below I’ll show you the seven steps and code that I used to build my map. 
##Step 1: Import Libraries 

```python
import pandas as pd 
import requests 
import xlrd 
import folium 
from folium.plugins import MarkerCluster
import wget 
import webbrowser
```



    
