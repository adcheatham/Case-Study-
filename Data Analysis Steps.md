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
### Set Working Directory
we will be setting the working directory to the path of the folder we created for it.
```{r}
setwd("/Users/litab/Documents/202207–202307_Cyclistic")
```

### Import and Merge
```{r}
csv_files <- list.files(path ="/Users/litab/Documents/202207–202307_Cyclistic" , recursive = TRUE, full.names=TRUE)

all_trips <- do.call(rbind, lapply(csv_files, read.csv))
```
### Display output
```{r}
str(all_trips)
```

### Rename 
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

### Finding Duplicates and drop NAs & missing Values
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

### Broad Overview of a data frame
```{r}
str(all_trips)
```

### Convert time into datetime format
```{r}
all_trips$start_time= ymd_hms(all_trips$start_time)
all_trips$end_time= ymd_hms(all_trips$end_time)
```

### Creating new columns needed for analysis
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

### Again take broad view of data
```{r}
skim(all_trips)
```
The ride lengths are also in negative and some values are quite high. Let's take a closer look.

```{r}
# See sorted ride_length data(first 30 rows) in ascending and descending order
all_trips %>%
  select(ride_length, start_station, end_station) %>%
  arrange(desc(ride_length)) %>%
  slice(1:30)

all_trips %>%
  select(ride_length, start_station, end_station) %>%
  arrange(ride_length) %>%
  slice(1:30)
```
Some ride lengths are quite high, it seems they are the testing bikes used at research facilities. Some ride lengths are less than a minute and may be due to lock opening and closing errors. Also, some ride lengths are negative. We have to filter out such data. 

### Let's see the percentile breakage.
```{r}
percentiles <- quantile(all_trips$ride_length, probs = seq(0, 1, 0.01), na.rm = TRUE)
data.frame(percentiles)
```
The values of 0% is -55 minutes.The difference between values of 1% and 100% is about 55944 minutes (almost 39 days), which is not realistic; while the difference between values of 1% and 99% is about 130 minutes, which makes much more sense. Based on this, in this statistical analysis of ride length, the value of 100% will be considered as outlier. Also, negative values should be dropped.

```{r}
all_trips <- all_trips[!(all_trips$ride_length <= 1 |
                         all_trips$ride_length >= percentiles[100]),]
```

```{r}
summary(all_trips)
```

### Export cleaned data to a csv file
```{r}
write.csv(all_trips, "all_trips_2021.csv")
```
It seems good to go for analysis part now.

## Analyze

### 1. Rides distribution
### - By type of user
```{r}
all_trips %>%
  group_by(user_type) %>%
  summarize(count = length(user_type), '%Rides' = (length(user_type) / nrow(all_trips)) * 100 )
```
```{r}
all_trips %>%
  ggplot(aes(user_type, fill = user_type)) + geom_bar() + labs(x= "Type of User", y= "Number of rides", title = "Rides Distribution By User Type")
```
### - By month
```{r}
all_trips %>%
  group_by(user_type, month) %>%
  summarise(number_of_ride = n(),
            .groups = "drop")
  all_trips %>%
  group_by(user_type, month) %>%
  summarise(number_of_ride = n(),
            .groups = "drop")%>%
  ggplot(aes(month, number_of_ride, fill = user_type)) +
  geom_col(position = "dodge") +
  labs(title = "The number of rides by month", x = "Month", y = "Number of rides")
```
### - By day of the month
```{r}
all_trips %>%
  group_by(user_type, day) %>%
  summarise(number_of_ride = n(),
            .groups = "drop")
```


```{r}
all_trips %>%
  group_by(user_type, day) %>%
  summarise(number_of_ride = n(),
            .groups = "drop")%>%
  ggplot(aes(day, number_of_ride, color = user_type)) +
   geom_line() +
  geom_point() +
  scale_y_continuous(labels = scales::label_number_si(),
                     breaks = seq(0, 100000, 10000)) +
  scale_x_continuous(breaks = seq(1, 31, 1)) +
  labs(title = "The number of rides by day of the month", x = "Day of the month", y = "Number of rides")
```

### - By day of the week
```{r}
all_trips %>%
  group_by(user_type, day_of_Week) %>%
  summarise(number_of_ride = n(),
            .groups = "drop")

 all_trips %>%
  group_by(user_type, day_of_Week)%>%
  summarise(number_of_ride = n(),
            .groups = "drop")%>%
  ggplot(aes(day_of_Week, number_of_ride, fill = user_type)) +
  geom_col(position = "dodge") +
  labs(title = "The number of rides by Weekday", x = "Weekday", y = "Number of rides")
```
### - By hour of the day
```{r}
all_trips %>% 
  mutate(hour = hour(start_time)) %>%
  group_by(user_type, hour) %>%
  summarise(no_of_rides = n(),
            .groups = "drop") %>%
  ggplot(aes(hour, no_of_rides, color = user_type)) +
  geom_line() +
  geom_point() +
  scale_y_continuous(labels = scales::label_number_si(),
                     breaks = seq(0, 300000, 25000)) +
  scale_x_continuous(breaks = seq(0, 23, 1)) +
  labs(title = "The number of rides by hour of the day", x = "Hour of the day", y = "Number of rides")
```
### - By hour of the day divided by days of the week
```{r}
all_trips %>% 
  mutate(hour = hour(start_time)) %>%
  group_by(user_type, hour, day_of_Week) %>%
  summarise(no_of_ride = n(),
            .groups = "drop") %>%
  ggplot(aes(hour, no_of_ride, color = user_type)) +
  geom_line() +
  geom_point() +
  scale_y_continuous(labels = scales::label_number_si(),
                     breaks = seq(0, 50000, 10000)) +
  scale_x_continuous(breaks = seq(0, 23, 1)) +
  labs(title = "The number of rides by hour divided by weekday", x = "Hour of the day", y = "Number of rides") +
  facet_wrap(~ day_of_Week)
```
### - Plot on same graph to compare weekday and weekends
```{r}
all_trips %>% 
  mutate(hour = hour(start_time),
         day_type = ifelse(day_of_Week %in% c("Sat", "Sun"), "Weekend", "Weekday")) %>%
  group_by(user_type, hour, day_type) %>%
  summarise(no_of_ride = n(),
            .groups = "drop") %>%
  ggplot(aes(hour, no_of_ride, color = user_type)) +
  geom_line() +
  geom_point() +
  scale_y_continuous(labels = scales::label_number_si(),
                     breaks = seq(0, 230000, 25000)) +
  scale_x_continuous(breaks = seq(0, 23, 1)) +
  labs(title = "The number of rides by hour divided by weekday and weekend", x = "Hour of the day", y = "Number of rides") +
  facet_wrap(~ day_type)
```

### 2. The summary statistics on the ride length
```{r}
#user-defined function for mode
Mode <- function(x) {
  ux <- unique(x)
  ux[which.max(tabulate(match(x, ux)))]
}
#Summary
print("Ride Length Summary")
all_trips %>% 
  summarize(mean = mean(ride_length),
            sd = sd(ride_length),
            min = min(ride_length),
            median = median(ride_length),
            max = max(ride_length),
            mode = Mode(ride_length))
```
### - group by user_type
```
all_trips %>%
  group_by(user_type) %>%
  summarize(mean = mean(ride_length),
            sd = sd(ride_length),
            min = min(ride_length),
            median = median(ride_length),
            max = max(ride_length),
            mode = Mode(ride_length))
```
### - View Distribution of ride length using box plots
``` all_trips %>%
  ggplot(aes(user_type, ride_length, fill = user_type)) +geom_boxplot() +
           scale_y_continuous(breaks = seq(0, 130, 10)) +
           labs(title = "Distribution of ride length", x = "User Type", y = "Length (min)")
```
### - The average ride length by day
```{r}
all_trips %>%
  group_by(user_type, day) %>%
  summarize(mean = mean(ride_length),
            .groups = "drop") %>%
arrange(day)
```

```{r}
all_trips %>%
  group_by(user_type, day) %>%
  summarize(mean = mean(ride_length),
            .groups = "drop") %>%
ggplot(aes(day, mean, color = user_type)) +
  geom_line() +
  geom_point() +
  scale_y_continuous(breaks = seq(0, 28, 2)) +
  scale_x_continuous(breaks = seq(1, 31, 1)) +
  labs(title = "The average ride length by day", x = "Day", y = "Average (min)")
```

### - The average ride length by month
```{r}
all_trips %>% 
  group_by(user_type, month) %>%
  summarize(mean = mean(ride_length),
            .groups = "drop") %>%
  ggplot(aes(month, mean, fill = user_type)) +
  geom_col(position = "dodge") +
  scale_y_continuous(breaks = seq(0, 28, 2)) +
  labs(title = "The average ride length by month", x = "Month", y = "Average (min)")
```

### - The average ride length by days of the week
```{r}
all_trips %>% 
  group_by(user_type, day_of_Week) %>%
  summarize(mean = mean(ride_length),
            .groups = "drop") %>%
  arrange(day_of_Week)
```

```{r}
all_trips %>% 
  group_by(user_type, day_of_Week) %>%
  summarize(mean = mean(ride_length),
            .groups = "drop") %>%
  ggplot(aes(day_of_Week, mean, fill = user_type)) +
  geom_col(position = "dodge") +
  scale_y_continuous(breaks = seq(0, 28, 2)) +
  labs(title = "The average ride length by days of the week", x = "Days of the week", y = "Average (min)")
```
### - The average ride length by hour of the day
```{r}
all_trips %>% 
  mutate(hour = hour(start_time)) %>%
  group_by(user_type, hour) %>%
  summarize(mean = mean(ride_length),
            .groups = "drop") %>%
  arrange(hour)
```

```{r}
all_trips %>% 
  mutate(hour = hour(start_time)) %>%
  group_by(user_type, hour) %>%
  summarize(mean = mean(ride_length),
            .groups = "drop") %>%
  ggplot(aes(hour, mean, color = user_type)) +
  geom_line() +
  geom_point() +
  scale_y_continuous(breaks = seq(0, 28, 2)) +
  scale_x_continuous(breaks = seq(0, 23, 1)) +
  labs(title = "The average ride length by hour of the day", x = "Hour of the day", y = "Average (min)")
```


### - By hour of the day divided by days of the week
```{r}
all_trips %>% 
  mutate(hour = hour(start_time),
         day_type = ifelse(day_of_Week %in% c("Sat", "Sun"), "Weekend", "Weekday")) %>%
  group_by(user_type, hour, day_type) %>%
  summarise(mean = mean(ride_length),
            .groups = "drop") %>%
  ggplot(aes(hour, mean, color = user_type)) +
  geom_line() +
  geom_point() +
  scale_y_continuous(breaks = seq(0, 30, 2)) +
  scale_x_continuous(breaks = seq(0, 23, 1)) +
  labs(title = "The average ride length by hour of the day divided by weekday and weekend", x = "Hour of the day", y = "Average (min)") +
  facet_wrap(~ day_type)
```
```{r}
all_trips %>% 
  mutate(hour = hour(start_time),
         day_type = ifelse(day_of_Week %in% c("Sat", "Sun"), "Weekend", "Weekday")) %>%
  group_by(user_type, hour, day_type) %>%
  summarise(mean = mean(ride_length),
            .groups = "drop") %>%
  ggplot(aes(hour, mean, color = user_type, shape = day_type)) +
  geom_line() +
  geom_point() +
  scale_y_continuous(breaks = seq(0, 30, 2)) +
  scale_x_continuous(breaks = seq(0, 23, 1)) +
  labs(title = "The average ride length by hour of the day divided by weekday and weekend", x = "Hour of the day", y = "Average (min)")
```
### 3. Analysis of Bike type

### - The number of bike type usage
```{r}
all_trips %>% 
  group_by(bike_type) %>%
  summarise(number_of_ride = n())
```
```{r}
all_trips %>% 
  group_by(bike_type) %>%
  summarise(number_of_ride = n()) %>%
  ggplot(aes(bike_type, number_of_ride, fill = bike_type)) +
  geom_col(position = "dodge", show.legend = FALSE) +
  scale_y_continuous(labels = scales::label_number_si(),
                     breaks = seq(0, 3000000, 1000000)) +
  labs(title = "The number of bike type usages", x = "Bike type", y = "Number of bike usages")
```
### - Bike type usage by user type
```{r}
all_trips %>% 
  group_by(user_type, bike_type) %>%
  summarise(number_of_ride = n(),
            .groups = "drop") %>%
  arrange(bike_type)
```

```{r}
all_trips %>% 
  group_by(user_type, bike_type) %>%
  summarise(number_of_ride = n(),
            .groups = "drop") %>%
  ggplot(aes(user_type, number_of_ride, fill = bike_type)) +
  geom_col(position = "dodge") +
  labs(title = "The number of bike type usages by user type", x = "User type", y = "Number of bike usages")
```
### - Bike type and average ride time
```{r}
all_trips %>% 
  group_by(bike_type) %>%
  summarise(mean = mean(ride_length),
            .groups = "drop") %>%
  arrange(bike_type)
```

```{r}
all_trips %>% 
  group_by(bike_type) %>%
  summarise(mean = mean(ride_length),
            .groups = "drop") %>%
  ggplot(aes(bike_type, mean)) +
  geom_col(position = "dodge") +
  labs(title = "The mean ride length by bike type", y = "Average Ride Length", x = "Bike Type")
```
### - Bike type and average ride time by user_type
```{r}
all_trips %>% 
  group_by(user_type, bike_type) %>%
  summarise(mean = mean(ride_length),
            .groups = "drop") %>%
  arrange(bike_type)
```
```{r}
all_trips %>% 
  group_by(user_type, bike_type) %>%
  summarise(mean = mean(ride_length),
            .groups = "drop") %>%
  ggplot(aes(bike_type, mean, fill = user_type)) +
  geom_col(position = "dodge") +
  labs(title = "The mean ride length by bike and user type", y = "Average Ride Length", x = "Bike Type")
```

### 4. Descriptive analysis on the station

### - Names of Unique Stations
```{r}
stations<- all_trips %>%
  gather(key, station_name, start_station, end_station) %>%
  distinct(station_name)
```

```{r}
print(paste("Number of stations are", nrow(stations)))
```
### - Most Popular stations
```{r}
pop_stations <- all_trips %>%
  gather(key, station_name, start_station, end_station) %>%
  group_by(station_name) %>%
  summarise(number_trip = n()/2) %>%
  arrange(desc(number_trip))

head(pop_stations,10)
```
```{r}
pop_stations %>%
  slice(1:10) %>%
  ggplot(aes(number_trip, reorder(station_name, number_trip))) +
  geom_col() +
  labs(title = "The top 10 most visited stations", x = "Number of trips", y = "Station name")
```
### - Most Popular stations by user type
```{r}
pu_station <- all_trips %>%
  gather(key, station_name, start_station, end_station) %>%
  group_by(user_type, station_name) %>%
  summarise(number_trip = n()/2,
            .groups = "drop_last") %>%
  arrange(desc(number_trip)) %>%
  slice(1:10)
```

```{r}
#Popular stations for casual riders
pu_station %>%
  head(10)
```


```{r}
pu_station %>%
  head(10) %>%
  ggplot(aes(number_trip, reorder(station_name, number_trip))) +
  geom_col() +
  scale_x_continuous(labels = scales::label_number_si(),
                     breaks = seq(0, 65000, 5000)) +
  labs(title = "The top 10 most visited stations by Casual riders", x = "Number of trips", y = "Station name")
```

```{r}
#Popular stations for member riders
pu_station %>%
  tail(10)
```


```{r}
pu_station %>%
  tail(10) %>%
  ggplot(aes(number_trip, reorder(station_name, number_trip))) +
  geom_col() +
  labs(title = "The top 10 most visited stations by member riders", x = "Number of trips", y = "Station name")
```
