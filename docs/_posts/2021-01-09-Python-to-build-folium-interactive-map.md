---
layout: post
title: "Building an Interactive UK Income Map"
date: 2021-01-09 16:10:00 -0000
categories: Python Folium
---
## Contents
* [Project Background](#background)
* [Output](#output)
* Process
	* [Step 1: Import Libraries](#step1) 
	* [Step 2: Download data required](#step2)
	* [Step 3: Merge the datasets and split the Local Authorities into income deciles](#step3)
	* [Step 4: Create our base Folium map and decile marker layers](#step4)
	* [Step 5: Create the boundary overlay](#step5)
	* [Step 6: Create the search functionality](#step6)
	* [Step 7: Create the Choropleth layer](#step7)
	* [Step 8: Save and open your map](#step8)
* [Final Code](#finalcode)
* [Remarks](#remarks)

## Project Background <a name="background"></a>
In this post I am going to show you how I built a Choropleth map using Folium for my first semester Python class. I love storytelling with data visualization, so I decided to look at using Python’s data visualization capabilities to display open source UK income data in a way that was interactive and easy to understand. 
I quickly discovered that Gross Disposable Household Income (GDHI) data by UK Local Authority was the most granular data publicly available, and that the Folium library/package would be the best way to approach this project. 
My objective was to produce a map that displayed:

* Markers displaying income information for each Local Authority
* Bottom 10% and top 10% of Local Authorities by income 
* Choropleth and boundary overlays
* A Local Authority search tool

[Folium documentation](https://python-visualization.github.io/folium/modules.html) and [Folium Plugins documentation](https://python-visualization.github.io/folium/plugins.html) are your best friends - there are so many more features available in Folium than are shown here! Below you can see the output that we'll create, and I’ll show you the eight steps and code that I followed. 

## Output <a name="output"></a>
<iframe src="/assets/folium_map.html" height="600" width="1000"></iframe>

# Step 1: Import Libraries <a name="step1"></a>

```python
import pandas as pd 
import requests 
import xlrd 
import folium 
from folium.plugins import MarkerCluster
import wget 
import webbrowser
```
# Step 2: Download data required <a name="step2"></a>
All the data required is available online from the Office for National Statistics (ONS), the nationally recognized institute that produces [official statistics in the UK](https://www.ons.gov.uk/). 
We will need three files:
* Co-ordinates to place the Local Authority markers on the map, available in *csv format, with 2019 the most recent available. [ONS Geography Portal](https://geoportal.statistics.gov.uk/datasets/local-authority-districts-december-2019-boundaries-uk-bfe-1/data) or [download file](https://opendata.arcgis.com/datasets/b3e3278842b6482ea53fb6314f52bbeb_0.csv?outSR=%7B%22latestWkid%22%3A27700%2C%22wkid%22%3A27700%7D). 
* Boundaries of the Local Authorities as a GeoJSON file. This file contains information on the geographical region covered by each Local Authority. This file was released December 2019. [Link to ONS site](https://geoportal.statistics.gov.uk/datasets/local-authority-districts-december-2019-boundaries-uk-bgc) or [download file](https://opendata.arcgis.com/datasets/0e07a8196454415eab18c40a54dfbbef_0.geojson). 
* Gross Disposable Household Income by Local Authority, available in xls* format. This data is split into 11 different excel files on the ONS website, so we will implement a loop to combine these files together. The latest data, released in 2020, contains GDHI for the years 1997-2018. [Find the files here](https://www.ons.gov.uk/economy/regionalaccounts/grossdisposablehouseholdincome/datasets/regionalgrossdisposablehouseholdincomebylocalauthoritiesbynuts1region).

```python
#1. Co-ordinates data(csv file)
url_coordinates_data = "https://opendata.arcgis.com/datasets/b3e3278842b6482ea53fb6314f52bbeb_0.csv?outSR=%7B%22latestWkid%22%3A27700%2C%22wkid%22%3A27700%7D"
resp = requests.get(url_coordinates_data)
coordinates_data=pd.read_csv(url_coordinates_data)

#2. Boundaries file (GeoJSON file) 
url_boundaries_data='https://opendata.arcgis.com/datasets/0e07a8196454415eab18c40a54dfbbef_0.geojson'
wget.download(url_boundaries_data, 'boundaries_data.geojson') 

#3. Income Data 
region_list=['cnortheast', 'dnorthwest','eyorkshireandthehumber', 'feastmidlands', 'gwestmidlands', 'heastofengland', 'ilondon', 'jsoutheast', 'ksouthwest', 'lwales', 'mscotland', 'nnorthernireland'] #list of all the NUTS regions in the UK
income_data=pd.DataFrame() #create empty dataframe that will contain all 11 datasets 
    
for j in region_list:
    if j=='eyorkshireandthehumber': #this region has a different URL structure from the others 
        url_income_data='https://www.ons.gov.uk/file?uri=%2feconomy%2fregionalaccounts%2fgrossdisposablehouseholdincome%2fdatasets%2fregionalgrossdisposablehouseholdincomebylocalauthoritiesbynuts1region%2fukeyorkshireandthehumber/regionalgrossdisposablehouseholdincomelocalauthorityukeyorksandhumber.xls'
    else:
        url_income_data=f'https://www.ons.gov.uk/file?uri=%2feconomy%2fregionalaccounts%2fgrossdisposablehouseholdincome%2fdatasets%2fregionalgrossdisposablehouseholdincomebylocalauthoritiesbynuts1region%2fuk{j}/regionalgrossdisposablehouseholdincomelocalauthorityuk{j}.xls'
    resp = requests.get(url_income_data) #retrieve data from HTML
    income_data_region=pd.read_excel(xlrd.open_workbook(file_contents=resp.content), sheet_name=3, header=1) #reads data from 4th sheet and takes 2nd row as column headers
    income_data_region.columns = income_data_region.columns.astype(str) #convert all columns names to strings which will cause less formatting issues when joining
    income_data = pd.concat([income_data, income_data_region]) #add datasets together
```
# Step 3: Merge the datasets and split the Local Authorities into income deciles <a name="step3"></a>
We merge the co-ordinates data with the income data so we work with just one data set throughout the rest of the steps. We will join the income and co-ordinates data by Local Authority Code. We also need to clean the data a bit.   

```python
income_data=income_data.rename(columns={"20182": "Y2018"}) #rename the column as the superscript in excel converted column name to 20182. The column name also needs to start with a string when we use pd.qcut below          
income_coordinates_merge=income_data.join(coordinates_data[['LAD19CD','LAD19NM', 'LAT', 'LONG']].set_index('LAD19CD'), on='LAD code', how='left')#merge datasets 
income_coordinates_data=income_coordinates_merge.dropna() #delete rows that contain no values, as we got lots of null values after joining
income_coordinates_data['Decile']=pd.qcut(income_coordinates_data['Y2018'],q=10, labels=['Lowest Decile', '2nd Decile', '3rd Decile', '4th Decile', '5th Decile', '6th Decile', '7th Decile', '8th Decile', '9th Decile', 'Highest Decile'])#creates a new column that assign the Local Authority its decile based on the income value. 
```
If you want to understand how qcut works, this table shows us that our Local Authorities have been divided into 10 (roughly) equal groups, where 'Lowest Decile' gives us the bottom 10% of Local Authorities by Gross Disposable Household Income:     

# Step 4: Create our base Folium map and decile marker layers <a name="step4"></a>
This step allows us to filter Local Authorities by deciles on our map, using the column 'Decile' that we created above. To create controllable layers in Folium, you have to use the FeatureGroup *. The code loops through the 10 groups that we created in Step 2, and creates a detailed marker for each Local Authority and groups these together as a layer. 
I also used *MarkerCluster here as the map gets overcrowded with markers really quick, making it nearly impossible to see the underlying map! 

```python
folium_map = folium.Map(location=[54.54, -1.21], zoom_start=5, min_zoom =5)#create base folium map

for grp_name, df_grp in income_coordinates_data.groupby('Decile'):
    feature_group=folium.FeatureGroup(grp_name, show=False)
    marker_cluster = MarkerCluster().add_to(feature_group) #add clusters for markers
    for row in df_grp.itertuples():
        folium.Marker([row.LAT, row.LONG],             
        popup='<b>Local Authority: </b>{} <br> <br> <b>GDHI (£ million)</b>: {}'.format(row.LAD19NM , row.Y2018), # pop-up label for the marker
        tooltip = "Click for more",
        icon=folium.Icon()
      ).add_to(feature_group).add_to(marker_cluster)
    feature_group.add_to(folium_map) 
```
# Step 5: Create the boundary overlay <a name="step5"></a>
This is more of a nice to have than a necessity. 

If you decide you don’t want to see the Choropleth overlay, but still want to see the boundary outlines of the Local Authorities then this overlay does just that.  

```python
style={'fillColor': '#00000000', 'color': '#000000', "weight": 1} 
geojson_obj=folium.GeoJson(
    'boundaries_data.geojson',
    style_function=lambda x:style,
    control=True, #show control in panel
    name='Local Authority Boundaries'
).add_to(folium_map) #create an object to be used in the search tool

```
# Step 6: Create the search functionality <a name="step6"></a>
This is probably my favourite functionality of the map! 

This makes it super easy to search for any Local Authority. For example, as you type in the search bar for ‘Manchester’, it lists suggestions such as ‘Mansfield’. It then zooms in on Manchester, just like Google Maps. The only downside is that if you want to see the information marker for Manchester, the decile that it is in must be ticked on the control pane
```python
folium.plugins.Search(
    layer=geojson_obj,
    search_label='LAD20NM',
    placeholder="Search for a Local Authority (e.g Manchester)",
    geom_type="Line", #zooms in when selecting a Local Aurthority
    position='topright'
).add_to(folium_map)
```
# Step 7: Create the Choropleth layer <a name="step7"></a>
This is where the magic happens! The Choropleth plugin* Will fill the Local Authority with a colour based on its GDHI value, where a darker colour indicates higher GDHI and vice versa, allowing us to spot instantly which Local Authorities have the highest or lowest GDHI.  
The default number of colours on the legend is 5, but I’ve increased it to 9 to get a wide spread of colours.
```python
folium.Choropleth(
     geo_data= 'boundaries_data.geojson',
     data=income_coordinates_data, 
     columns=['LAD code', 'Y2018'], # LAD code is here for matching the geojson zipcode, 2018 is the column that changes the color of zipcode areas
     key_on='feature.properties.LAD20CD', # this path contains zipcodes in str type, this zipcodes should match with our 'LAD code' column
     fill_color='BuPu', #change colour based on the list we created
     bins=9 , #divide the legend into maximum 9 different colours, evenly split by the values. Not to be confused with Deciles!
     fill_opacity=1, 
     line_opacity=1,
     legend_name='Gross Disposable Household Income (£ million) 2018', 
     highlight=True,
     name='Choropleth Overlay',
     nan_fill_color='white' #if can't find LA boundary, default to white- only 1 LA that has changed from 2018 to 2020 (split from 1 to 3 LAs: Chiltern, Wycombe, Aylesbury Vale)
     ).add_to(folium_map)

folium.LayerControl(collapsed=False).add_to(folium_map)#create a panel that allows the user to select between layers
```
# Step 8: Save and open your map <a name="step8"></a>
The map is saved in HTML format, and will open automatically on your default browser. Voilà!
```python
folium_map.save('folium_map.html') #save as HTML
webbrowser.open('folium_map.html') #open HTML
```
## Final Code <a name="finalcode"></a>
Can be found at my [GitHub Repository]()

## Remarks <a name="remarks"></a>
As a first time Python developer*, I was very impressed at Python’s data wrangling and data visualization capabilities. I was amazed at the powerful infographic?s you can achieve with without paying for proprietary data visualization software and only using open data! I hope this helps if you are looking to produce a similar map. 

Feel free to contact me if you are looking for someone to build something similar for your business.    

