# Grand Rapids Nonprofit Clusters
Francisco Santamarina  
November 8, 2017  



### Data Acquisition and Preparation
Nathan Grasse and Jesse Lecy collected a large amount of efiler data and shared the datasets with the attendees of the 2017 West Coast Data Conference, held at the University of Washington in Seattle, WA. 

The 2013 dataset that they provided was used to pull data on nonprofits operating in Grand Rapids, MI during 2013. Once accessed, it must be cleaned, geocoded, and aggregated to show the distribution of nonprofits in relation to the city center of Grand Rapids, MI. 

**Please note**: This section walks through the steps of cleaning the data. The primary source dataset was almost 500MB, resulting in a very slow process. Please keep the size of the dataset in mind when exploring the data! The dataset used below was the 2013 dataset provided at the West Coast Data Conference just for Michigan nonprofits.

*Thanks to Christine Brown for her code written in Spring 2017 at the Maxwell School*.

```r
#Load packages
library( dplyr )
library( geojsonio )
library( ggmap )
library( maps )
library( maptools )
library( raster )
library( rgdal )
library( rgeos )
library( sp )
```

```r
#Set your working directory
# setwd( "    " )

#Download the Michigan 2013 dataset into R
efilers2013 <- read.csv( "././990 Efiler Data from WestCoast/990EFILE-2013_MI only.csv" )

#Check for unique combinations of city names, in case "Grand Rapids" is misspelled
cityNames <- grep( "^GRAND R", unique( efilers2013$CITY ), value = T, ignore.case = T )

#Subset the dataset to only contains values with some version of "Grand Rapids" as the city
dat <- efilers2013[ efilers2013$CITY %in% cityNames, ]

#Remove superfluous variables, since the dataset has already been subsetted
rm(cityNames)
rm(efilers2013)

#Write data as a CSV for future access
write.csv( dat, file = "Grand Rapids 2013 NPO Efilers.csv", row.names = FALSE)

#Clean data for geocoding
npo_addresses <- dat[ , c("ADDRESS", "STATE", "ZIP") ]
npo_addresses$ADDRESS <- gsub( ",", "", npo_addresses$ADDRESS ) #removes commas in addresses
npo_addresses$ADDRESS <- gsub( "\\.", "", npo_addresses$ADDRESS ) #removes periods in addresses

#Combine the strings in this order: Address, City, State, Zip. Each is separated with a comma and a space
geocoding <- paste( npo_addresses$ADDRESS, "GRAND RAPIDS", npo_addresses$STATE, npo_addresses$ZIP, sep=", " )

#Geocode
npo_coordinates <- suppressMessages( geocode( geocoding , messaging = F ) )

#Add longitude and latitude to efiler dataset
dat <- cbind( dat, npo_coordinates )
write.csv( dat, file = "Grand Rapids 2013 NPO Efilers with geographic coordinates.csv", row.names = FALSE)

#Remove unnecessary dataframes and vectors from the R environment
rm(npo_addresses)
```

### Data Visualization: NPOs Around City Hall in 2013
A common metric used in analyzing civil society health is nonprofit density within a metropolitan area. The map below shows two circles centered on Grand Rapids' City Hall, located at 42.9692626 N and 85.6715216 W according to Google Maps. The circles help us see visually that many nonprofits in Grand Rapids are within a half-mile of City Hall. 
An interesting idea for future analysis would be to look at the Metropolitan Statistical Area of [Grand Rapids](http://proximityone.com/metros/2013/cbsa24340.htm), and using zipcodes as the filter instead of the city of "Grand Rapids". This markdown file looks solely at the neighborhoods that compose Grands Rapids proper. 

```r
#Read the geoJSON file into R
gr_nhoods <- geojson_read( "http://data.grcity.us/storage/f/2014-03-04T22%3A22%3A42.217Z/neighborhoods.json", method="local", what="sp" )
gr_nhoods <- spTransform( gr_nhoods , CRS( "+proj=longlat +datum=WGS84" ) )

#Read in your cleaned data
dat <- read.csv( "https://raw.githubusercontent.com/fjsantam/ARNOVA-2017-NPOmap/master/Grand%20Rapids%202013%20NPO%20Efilers%20with%20geographic%20coordinates.csv" )
npo_coordinates <- dat[ , c("lon", "lat" ) ]

#Remove coordinates with NA values
npo_coordinates <- filter( npo_coordinates , !is.na( lon ) )
npo_coordinates <- filter( npo_coordinates , !is.na( lat ) )

#Assign spatial points
npo_coordinates_SP <- SpatialPoints( npo_coordinates , proj4string = CRS("+proj=longlat +datum=WGS84" ) )

#Subset your spatial points to those within the limits of the geoJSON map
npo_coordinates_SP <- npo_coordinates_SP[ gr_nhoods ]

#Assign the location of City Hall
lon <- -85.6715216
lat <- 42.9692626
CityHall <- as.data.frame( cbind ( lon, lat ) )
CityHall <- SpatialPointsDataFrame( CityHall, CityHall, proj4string = CRS( "+proj=longlat +datum=WGS84" ) )

#Create buffers
gr_outline <- gBuffer( gr_nhoods , width = .000 , byid = F )
buff_half <- gBuffer( CityHall , width = .0095 , byid = F )
buff_half_clipped <- gIntersection( gr_outline , buff_half , byid = TRUE , drop_lower_td = T )
buff_one <- gBuffer( CityHall , width = .019 , byid = F )
buff_one_clipped <- gIntersection( gr_outline , buff_one , byid = TRUE , drop_lower_td = T )

#Plot buffers
par( mar = c( 0 , 0 , 1 , 0 ) )
plot( gr_nhoods , col = "gray89" , main = "Nonprofits in Grand Rapids in 2013 in relation to City Hall" )
plot( buff_one_clipped , col = rgb( 10 , 95 , 193 , 40 , maxColorValue = 255 ) , border = F , add = T )
plot( buff_half_clipped , col = rgb( 10 , 95 , 193 , 70 , maxColorValue = 255 ) , border = F , add = T )
points( npo_coordinates_SP , pch = 21 , col = "#dd5e04", bg = alpha( "#dd7804", 0.5), cex = 1.5, lwd = 1.5 )
points( CityHall$lon, CityHall$lat, col = "#d10404", cex = 2.5, pch = 13, lwd = 2.8 )
map.scale( x = -85.75183 , y = 42.895 , metric = F , ratio = F , relwidth = 0.15 , cex = 1 )

#Add legend
legend( x = -85.75183 , y = 42.93 , pch = 20 , pt.cex = 1.4 , cex = .8 , xpd = NA , bty = "n" , 
        inset = -.01 , legend = c( "City Hall", "Nonprofit" , "1/2 Mile Radius" , "1 Mile Radius" ) , 
        col = c( "#d10404", "#dd7804" , rgb( 10 , 95 , 193 , 94 , maxColorValue = 255 ) , 
                 rgb( 10 , 95 , 193 , 74 , maxColorValue = 255 ) ) )
```

![](Grand_Rapids_clusters_171108_ARNOVA_Hotel_files/figure-html/unnamed-chunk-3-1.png)<!-- -->

### Data Visualization: NPOs Around Amway Grand Plaza Hotel in 2013
A common metric used in analyzing civil society health is nonprofit density within a metropolitan area. The map below shows two circles centered on Grand Rapids' Amway Grand Plaza Hotel, a 5-minute walk from City Hall. It is located at 42.966833 N and 85.6731046 W according to Google Maps. The circles help us see visually that many nonprofits in Grand Rapids are within a half-mile of Amway Grand Plaza, and grants us an insight into the past of the location of the 2017 ARNOVA conference.


```r
#Read the geoJSON file into R
gr_nhoods <- geojson_read( "http://data.grcity.us/storage/f/2014-03-04T22%3A22%3A42.217Z/neighborhoods.json", method="local", what="sp" )
gr_nhoods <- spTransform( gr_nhoods , CRS( "+proj=longlat +datum=WGS84" ) )

#Read in your cleaned data
dat <- read.csv( "https://raw.githubusercontent.com/fjsantam/ARNOVA-2017-NPOmap/master/Grand%20Rapids%202013%20NPO%20Efilers%20with%20geographic%20coordinates.csv" )
npo_coordinates <- dat[ , c("lon", "lat" ) ]

#Remove coordinates with NA values
npo_coordinates <- filter( npo_coordinates , !is.na( lon ) )
npo_coordinates <- filter( npo_coordinates , !is.na( lat ) )

#Assign spatial points
npo_coordinates_SP <- SpatialPoints( npo_coordinates , proj4string = CRS("+proj=longlat +datum=WGS84" ) )

#Subset your spatial points to those within the limits of the geoJSON map
npo_coordinates_SP <- npo_coordinates_SP[ gr_nhoods ]

#Assign the location of City Hall
lon <- -85.67310459999999
lat <- 42.966833
Amway <- as.data.frame( cbind ( lon, lat ) )
Amway <- SpatialPointsDataFrame( Amway, Amway, proj4string = CRS( "+proj=longlat +datum=WGS84" ) )

#Create buffers
gr_outline <- gBuffer( gr_nhoods , width = .000 , byid = F )
buff_half <- gBuffer( Amway , width = .0095 , byid = F )
buff_half_clipped <- gIntersection( gr_outline , buff_half , byid = TRUE , drop_lower_td = T )
buff_one <- gBuffer( Amway , width = .019 , byid = F )
buff_one_clipped <- gIntersection( gr_outline , buff_one , byid = TRUE , drop_lower_td = T )

#Plot buffers
par( mar = c( 0 , 0 , 1 , 0 ) )
plot( gr_nhoods , col = "gray89" , main = "Nonprofits in Grand Rapids in 2013 in relation to Amway Grand Plaza" )
plot( buff_one_clipped , col = rgb( 10 , 95 , 193 , 40 , maxColorValue = 255 ) , border = F , add = T )
plot( buff_half_clipped , col = rgb( 10 , 95 , 193 , 70 , maxColorValue = 255 ) , border = F , add = T )
points( npo_coordinates_SP , pch = 21 , col = "#dd5e04", bg = alpha( "#dd7804", 0.5), cex = 1.5, lwd = 1.5 )
points( Amway$lon, Amway$lat, col = "#d10404", cex = 2.5, pch = 13, lwd = 2.8 )
map.scale( x = -85.75183 , y = 42.895 , metric = F , ratio = F , relwidth = 0.15 , cex = 1 )

#Add legend
legend( x = -85.79183 , y = 42.93 , pch = 20 , pt.cex = 1.4 , cex = .8 , xpd = NA , bty = "n" , 
        inset = -.01 , legend = c( "Amway Grand Plaza Hotel", "Nonprofit" , "1/2 Mile Radius" , "1 Mile Radius" ) , 
        col = c( "#d10404", "#dd7804" , rgb( 10 , 95 , 193 , 94 , maxColorValue = 255 ) , 
                 rgb( 10 , 95 , 193 , 74 , maxColorValue = 255 ) ) )
```

![](Grand_Rapids_clusters_171108_ARNOVA_Hotel_files/figure-html/unnamed-chunk-4-1.png)<!-- -->
