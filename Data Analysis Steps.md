## Data Analysis Steps

## 1. ASK 
**Business Task:** Design marketing strategies aimed at converting casual riders into annual members. 

**Moreno has assigned you to answer:** How do annual members and casual riders use Cyclistic bikes differently?

### Stakeholders
**Lily Moreno** - The director of marketing and your manager. Moreno is responsible for the development of campaigns and
initiatives to promote the bike-share program. These may include email, social media, and other channels.

**Cyclistic executive team** 

**Cyclistic marketing analytics team**

## 2. Prepare 
To conduct a thorough analysis, we have to obtain the Cyclistic historical [data trips](https://divvy-tripdata.s3.amazonaws.com/index.html).
The data types are appropriate and consistent throughout and is in .csv format from July 2022 to July 2023. This data has been made available by Motivate International Inc. under this [license](https://divvybikes.com/data-license-agreement).

## 3. Process
To clean, analyze, and aggregate the large amount of monthly data stored in the folder, we will be using RStudio 
First we will start but loading our packages. You may have to install tidyverse first.

### LOAD PACKAGES
```{r}
library(tidyverse) 
library(lubridate) 
library(dplyr)     
library(janitor)      
library(ggplot2)
```
## Set Working Directory
we will be setting the working directory to the path of the folder we created for it.
```{r}
setwd("/Users/litab/Documents/202207–202307_Cyclistic")
```

## Import and Merge
```{r}
csv_files <- list.files(path ="/Users/litab/Documents/202207–202307_Cyclistic" , recursive = TRUE, full.names=TRUE)

all_trips <- do.call(rbind, lapply(csv_files, read.csv))
```
## Display output
```{r}
str(all_trips)
```

## Rename 
```{r}
all_trips <- rename(all_trips,
                    trip_id = ride_id,
                    bike_type = rideable_type,
                    start_time = started_at,
                    end_time = ended_at,
                    start_station = start_station_name,
                    ss_id = start_station_id,
                    end_station = end_station_name,
                    es_id = end_station_id,
                    user_type = member_casual)
```

## Finding Duplicates and drop NAs & missing Values
```{r}
anyDuplicated(all_trips$trip_id)
original_rows <- nrow(all_trips)
all_trips <- drop_na(all_trips)
all_trips <- all_trips[!(all_trips$start_station == "" |
                         all_trips$ss_id == "" |
                         all_trips$end_station == "" |
                         all_trips$es_id == "" ),]
print(paste("Removed", original_rows - nrow(all_trips), " rows"))
```

## Broad Overview of a data frame
```{r}
str(all_trips)
```

## Convert time into datetime format
```{r}
all_trips$start_time= ymd_hms(all_trips$start_time)
all_trips$end_time= ymd_hms(all_trips$end_time)
```

## Creating new columns needed for analysis
```{r}
#Calculate ride length of all trips
all_trips$ride_length <- as.double(difftime(all_trips$end_time, all_trips$start_time, units = "mins"))
# Add columns that list the date, month, day, and year of each ride
all_trips$date <- as.Date(all_trips$start_time) #The default format is yyyy-mm-dd
all_trips$month <- month(all_trips$date, label = TRUE)
all_trips$day <- mday(all_trips$date)
all_trips$year <- year(all_trips$date)
all_trips$day_of_Week <- wday(all_trips$date, label = TRUE)
```

