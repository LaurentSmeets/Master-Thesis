This file contains all the code, in chronological order, to reproduce my Thesis. Due to privacy reasons and the rules of Statistics Netherlands, no raw input data is available. However, the folder structure and names of the files that are needed are present.

### Packages and working directory

``` r
library(stringr)
library(haven)
library(data.table)
library(lubridate)
library(forcats)
library(viridis)
library(knitr)
library(geosphere)
library(zoo)
library(argosfilter)
library(trajr)
library(sp)
library(hutils)
library(imputeTS)
library(foreign)
library(nnet)
library(reshape2)
library(maptools)
library(rgdal)
library(tidyverse)
library(reshape2)
library(PerformanceAnalytics)
library(ggforce)
library(corrplot)
library(GGally)
library(ggcorrplot)
library(userfriendlyscience)
library(rcompanion)
library(ggthemes)
library(gganimate)
library(tmap)
library(tmaptools)
library(sf)
library(xtable)
library(OpenStreetMap)
library(jsonlite)
library(proj4)
library(caret) 
library(scales)
library(ggrepel)
library(lime)
library(vip)
library(pdp)
library(jtools)
library(stargazer)
library(skimr)
library(broom)

setwd("xxxx") # set top level directory
```

This file contains the code to match the data in such a way that you get a dataset that contains only data for the GPS points that belong to a trip that has been coded by the user. the end file thus has a lat, lon, time stamp, transport mode, trip.id, device.id, in which every row is a unique combination of time stamp and lat/lon.

### set up folder structure

To run this code the file structure needs to be such that there is an input folder in the working directory in which the data (gps, tracks, transport, etc.) are collected per day in a separate folder. There also needs to be a folder for all the code and a folder for the output.

``` r
list.dirs(full.names=F, recursive = F)
```

The contents of the input\_data are ordered as:

``` r
list.dirs(path ="./Imput_data", full.names=F, recursive = F)
```

and within the folders the structure looks like

``` r
list.files(path ="./Imput_data/2018-11-01", full.names=F, recursive = F)
```

### load transportmodes

This code loads in the transport modes for all the separate days and bind them into one file

``` r
# 1) list all the folders in the Imput_data folder
folders<-list.dirs(path ="./Imput_data",
                   full.names=F,
                   recursive = F)

# 2) create an empty list to store the transport modes in
transportmodelist <- list()

# 3) loop over the different folders and load in the transport modes
for(i in 1:length(folders)){
  transportmodelist[[i]]<-fread(paste0(getwd(), "/Imput_data/",
                                       folders[i], 
                                       "/transportation_modes.csv"))
}
# 4) bind the transport modes of different days together
transportmodes1101_0220 <-rbindlist(transportmodelist)
rm(transportmodelist)
```

The transport mode data now looks like this:

``` r
head(transportmodes1101_0220, 20)
```

There are quite some redundant columns/variables in the data that can be deleted.

``` r
transportmodes1101_0220[, c("walk", 
                            "run", 
                            "mobility_scooter", 
                            "car",
                            "bike",
                            "moped",
                            "scooter",
                            "motorcycle",
                            "train",
                            "subway",
                            "tram",
                            "bus",
                            "other",
                            "updated_at","deleted_at"):= NULL]
```

Now the data.table looks like this

``` r
head(transportmodes1101_0220, 20)
```

Further inspection of the variable active modes also indicates that there are respondents who did use the 'others' option in the active modes to indicate (technical) issues they had with the app or registration of the specific trip. It is important to filter out these entries and label them as a special category (for now called *User error*). To do this we create a string of 'problem' words and check of these word (or words similar to them) are in the active mode variable

``` r
problemwords <- c("Didnt go there",
                  "fout",
                  "dwaling",
                  "Incorrect",
                  "Klopt niet",
                  "Flauwekul",
                  "niet ",
                  "Onjuist",
                  "onterecht",
                  "NVT",
                  "n.v.t",
                  "probleem",
                  "geen",
                  "Verkeerd",
                  "verkeert",
                  "nooit",
                  "Goute",
                  "mis meting",
                  "No transport",
                  "Was gewoon thuis",
                  "Thuis",
                  "did not move",
                  "Onbekend",
                  "Miet weg",
                  "niet")
```

With this string we can now start to recode the string variable *active modes* into more useful variables, in the following steps

-   Delete all special characters and punctuation marks (except commas) (*active modes* -&gt; *mode*)
-   recode the trips with more than 1 transportation mode to be labelled *multi* (*mode* -&gt; *mode1*)
-   delete trips with no mode filled in. This can happen when the user does open the app to fill in a transport mode, but then closes the app without doing so.
-   create categories of transport modes that will be distinguished, and label all other transport modes as *other\_transport\_mode* (*mode1* -&gt; *mode2*)
-   relabel some very similar transport modes, transport modes entered with typos, and transport modes which have been entered uncapitalized or with an extra space. for example bromfiets becomes Scooter and te voet becomes Walk
-   certain categories get pulled together. For example CarPassenger, CarDriver, elektrische auto, Auto, busje all become car. similar groupings where formed for Scooter and Walk (*mode2* -&gt; *mode3*)
-   *active modes* entries with one of the `problemwords` or similar words where labeled as "User error"

``` r
transportmodes1101_0220 <- transportmodes1101_0220 %>%
  mutate(mode = str_replace_all(active_modes, "[^[abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ, :]]", "")) %>% #remove all the forward slashes, brackets and other special characters 
  mutate(mode = if_else(grepl(",$", mode),    # this part is needed to delete commas that are at the end of the string of transportmodes and not between transportmodes as seperator. 
                        gsub('.{1}$', "", mode),
                        mode)) %>%
  mutate(mode1 = ifelse(grepl(",",mode, fixed=T), "multi", mode))%>% # reclassify trips that have more than one mode into multi mode
  filter(mode1!="")%>% # filter out trips with no transportmodes. this can happen when someone opens the question, but does not add anything.
  mutate(mode2=ifelse(mode1 %in% c("Walk",
                                  "BikeNonElectric",
                                  "CarDriver",
                                  "CarPassenger",
                                  "Tram",
                                  "Train",
                                  "PublicBus",
                                  "BikeElectric",
                                  "Metro",
                                  "multi",
                                  "Van",
                                  "werkbus",
                                  "vrachtwagen",
                                  "Motorbike") | 
                        grepl("scooter", mode1, ignore.case=T)|
                        grepl("bromfiets", mode1, ignore.case=T)|
                        grepl("snorfiets", mode1, ignore.case=T)|
                        grepl("bestelbus", mode1, ignore.case=T)|
                        grepl("voet", mode1, ignore.case=T),
                      mode1, "other_transport_mode")) %>%
  # now we recode all motorized two-weelers into the category "Scooter", also those thathave to many spaces in the character string
  # with the exeption of a motor bike
  mutate(mode2 = ifelse(grepl("scooter", mode1, ignore.case=T), "Scooter", mode2))%>%
  mutate(mode2 = ifelse(grepl("te voet", mode1, ignore.case=T), "Te Voet", mode2))%>%
  mutate(mode2 = ifelse(grepl("bestelbus", mode1, ignore.case=T), "Bestelbus", mode2))%>%
  mutate(mode2 = ifelse(grepl("bromfiets", mode1, ignore.case=T), "Bromfiets", mode2))%>%
  mutate(mode3 = ifelse(grepl("bromfiets", mode1, ignore.case=T), "Scooter",
                               ifelse(grepl("snorfiets", mode1, ignore.case=T), "Scooter",mode2))) %>%
  # now we group any thing that includes "voet" or "wandelen" in the category "Walk"
  # and we group small 4-weeled vehicle such as "auto", "van", "busje", "werkbus" into the category "car"
  # also passenger and drivers of the car are seen as one category
  mutate(mode3 = ifelse(grepl("voet", mode1, ignore.case=T), "Walk",
                        ifelse(grepl("wandelen", mode1, ignore.case=T), "Walk",
                               ifelse(grepl("elektrische auto", mode1, ignore.case=T), "Car",
                                      ifelse(grepl("CarPassenger", mode1, ignore.case=T), "Car",
                                             ifelse(grepl("Auto", mode1, ignore.case=T), "Car",
                                                    ifelse(grepl("Van", mode1, ignore.case=T), "Car",
                                                           ifelse(grepl("busje", mode1, ignore.case=T), "Car",
                                                                  ifelse(grepl("werkbus", mode1, ignore.case=T), "Car",
                                                                      ifelse(grepl("bestelbus", mode1, ignore.case=T), "Car",
                                                                         ifelse(grepl("CarDriver", mode1, ignore.case=T), "Car",mode3))))))))))) %>%
  # we change the category "vrachtwagen" and anything similar to it to "lorry"
  mutate(mode3 = ifelse(grepl("vrachtwagen", mode1, ignore.case=T), "Lorry", mode3)) %>%
  mutate(mode3 = ifelse(grepl(paste(problemwords, collapse = "|"), mode1, ignore.case=T), "User error", mode3))
```

### load trips

to be able to match transport modes to gps coordinates, we need to first match transport modes to tracks, as recorded by the app. A trip is a time period between two stops and since these are recorded automatically by the app and not all users coded all trip, there are more trips than transport modes. In the data from the 1st of November until the 12th of December there are 438124 transport modes in the raw data and 1411503 trips. Actually there are a lot less that this, since many trips are recorded multiple times in the data file due to some technical issues,those issues are resolved and accounted for later.

``` r
trackslist <- list()
for(i in 1:length(folders)){
  trackslist[[i]]<-fread(paste0(getwd(), "/Imput_data/",
                                folders[i], 
                                "/tracks.csv"))
}

tracks1101_0220 <-rbindlist(trackslist)
rm(trackslist)

# delete unnecessary variables
tracks1101_0220[, c("updated_at","deleted_at"):= NULL]
```

The tracks data then looks like this:

``` r
head(tracks1101_0220, 20)
```

There are no transport modes add to the trip data yet.

### join trips and transport modes

It happens quite often that the data points can belong to different trips. This is because the app sometimes struggles to determine what the stop of a trip is. Secondly, when a user goes back in the app later to assign a transport mode to the trip a new with a new id is created. Considering I am interested in trips that have been coded, I use two filter rules. 1) trips with the same start time, but different end times I keep the trips with the latest end time. 2) trips with the same start time and same end time I keep the the trips that have a transport mode assigned to them. 3) trips with the same start time and same end time and have a transport mode assigned to them, I keep the trip that was created last.

In addition the code above already guaranteed that trips with the same that have been assigned multiple transport modes have been filtered out already.

``` r
# create a variable to join tracks to transport modes
setDT(transportmodes1101_0220)
transportmodes1101_0220[, matchid:=track_id]

# create a variable to join tracks to transport modes
setDT(tracks1101_0220)
tracks1101_0220[, matchid:=id]

#set times to actual times
transportmodes1101_0220[, timestamp := as_datetime(timestamp, tz= "Europe/Paris")]
transportmodes1101_0220[, created_at := as_datetime(created_at, tz= "Europe/Paris")]
tracks1101_0220[, start_time := as_datetime(start_time, tz= "Europe/Paris")]
tracks1101_0220[, end_time := as_datetime(end_time, tz= "Europe/Paris")]
tracks1101_0220[, created_at := as_datetime(created_at, tz= "Europe/Paris")]


joinedtrack.transport1101_0220<-left_join(tracks1101_0220, transportmodes1101_0220, by="matchid")
#joinedtrack.transport1101_1209<-left_join(tracks1101_1209, transportmodes1101_1209, by = c("id"= "track_id"))
setDT(joinedtrack.transport1101_0220)

# detele unnessary variables
joinedtrack.transport1101_0220[, c("created_at.y","device_id.y"):=NULL]

# only keep the unique entries
joinedtrack.transport1101_0220.unique <- unique(joinedtrack.transport1101_0220, by=c("device_id.x", "local_id", "start_time", "end_time","created_at.x", "track_id", "active_modes","mode1", "mode3"))

# filter only the location data of those respondents that at least filled in one transport mode 
# this is to keep file size limited
completed_atleast_one<-joinedtrack.transport1101_0220.unique%>%
  group_by(device_id.x)%>%
  summarize(tt=sum(!is.na(mode3)))%>%
  filter(tt!=0)
completed_atleast_one<-completed_atleast_one$device_id.x



# rename some variables in the tracks data
setDT(joinedtrack.transport1101_0220.unique)
setnames(joinedtrack.transport1101_0220.unique, "id.x", "TrackDataIdVar")
setnames(joinedtrack.transport1101_0220.unique, "created_at.x", "TrackDataCreatedAt")
setnames(joinedtrack.transport1101_0220.unique, "device_id.x", "device_id")

# delete some variables
#joinedtrack.transport1101_1209.unique[,created_at.x:= NULL,]
joinedtrack.transport1101_0220.unique[,local_id:= NULL,]
joinedtrack.transport1101_0220.unique[,track_id:= NULL,]
joinedtrack.transport1101_0220.unique[,id.y:= NULL,]


#filter data for respondents who did fill in at least one trip
#joinedtrack.transport1101_1209.unique.filtered<-joinedtrack.transport1101_1209.unique[device_id%in%completed_atleast_one,,]
#joinedtrack.transport1101_1209.unique.filtered[, matchid:=NULL]



# transform those tracks that are listed multiple times into 'multi' and delete double entries

doubles <- joinedtrack.transport1101_0220.unique%>%
  group_by(TrackDataIdVar)%>%
  summarize(n = n())%>%
  filter(n !=1) %>%
  select(TrackDataIdVar)

joinedtrack.transport1101_0220.unique <- joinedtrack.transport1101_0220.unique%>%
  mutate(mode3 =ifelse(TrackDataIdVar %in% doubles$TrackDataIdVar, "trip saved twice with multi modes", mode3))

joinedtrack.transport1101_0220.unique <- joinedtrack.transport1101_0220.unique %>%
  group_by(device_id, start_time) %>%
  mutate(end_time = max(end_time),
         mode3 = mode3[which.max(TrackDataIdVar)])

double.max.TrackDataIdVars <- joinedtrack.transport1101_0220.unique %>%
  group_by(device_id, start_time) %>%
  filter(TrackDataIdVar == TrackDataIdVar[which.max(TrackDataIdVar)])%>%
  group_by(TrackDataIdVar)%>%
  summarise(n= n())%>%
  filter(n>1)

joinedtrack.transport1101_0220.unique<- joinedtrack.transport1101_0220.unique %>%
  group_by(device_id, start_time) %>%
  filter(TrackDataIdVar == TrackDataIdVar[which.max(TrackDataIdVar)])%>%
  group_by(TrackDataIdVar) %>%
  filter(if(TrackDataIdVar %in% double.max.TrackDataIdVars$TrackDataIdVar) timestamp == timestamp[which.max(timestamp)] else TrackDataIdVar>0)

setDT(joinedtrack.transport1101_0220.unique)
joinedtrack.transport1101_0220.unique2 <- joinedtrack.transport1101_0220.unique[!is.na(mode3),,]

all_unique_tracks_1101_0220 <- joinedtrack.transport1101_0220.unique
all_unique_tracks_1101_0220_with_transportmode <- joinedtrack.transport1101_0220.unique2
```

### clean up work space 1 and save the end files

``` r
# delete the all the files not of interest:
saveRDS(all_unique_tracks_1101_0220_with_transportmode, "output_files/all_unique_tracks_1101_0220_with_transportmode.RDS")
saveRDS(all_unique_tracks_1101_0220, "output_files/all_unique_tracks_1101_0220.RDS")
saveRDS(completed_atleast_one, "output_files/completed_atleast_one.RDS")
rm(list=gdata::keep("all_unique_tracks_1101_0220_with_transportmode", "completed_atleast_one", "folders"))
```

Make some basic plots to visualize data.

``` r
ggplot(data = all_unique_tracks_1101_0220_with_transportmode)+
  geom_bar(aes(x=fct_infreq(mode3), 
               fill=fct_infreq(mode2)),
           col = "red")+
  theme(axis.text.x = element_text(angle = 90, hjust = 1))+
  scale_fill_discrete(name= "legend")

ggplot(data = filter(all_unique_tracks_1101_0220_with_transportmode, mode3=="multi" & mode3 !="User error"))+
  geom_bar(aes(x=fct_infreq(mode)))+
  coord_flip()
```

### combine GPS data with transpormode data

This code can be used to count the total number of gps data points.

``` r
totalrows <-c()
for(i in 1:length(folders)){
  totalrows[i] <-nrow(fread(paste0(getwd(), "/Imput_data/", folders[i], "/position_entries.csv"), select =1L))
} 
sum(totalrows)
```

here we join the GPS data with the transport mode data and create one big data frame with all GPS coordinates. Of course one could save a lot of time by doing this per day instead of at once, but then one would not capture the trips during midnight.

``` r
locationslist <- list()
for(i in 1:length(folders)){
  locations<-fread(paste0(getwd(), "/Imput_data/",
                                       folders[i], 
                                       "/position_entries.csv"))
  setDT(locations)
  #locations <- locations[device_id%in%completed_atleast_one,,]
  locations[, c("heading", "speed", "created_at", "distance_between_previous_position", "desired_accuracy"):=NULL]
  
  locations[, timestamp_location:=as_datetime(timestamp, tz= "Europe/Paris")] # assign a real time stamp
  locations[, timestamp_location_dummy:=timestamp_location ]
  
  setkey(locations, device_id, timestamp_location, timestamp_location_dummy)
  setkey(all_unique_tracks_1101_0220_with_transportmode, device_id, start_time, end_time)
  
  joined.data <- foverlaps(locations, all_unique_tracks_1101_0220_with_transportmode, 
                           by.x = c("device_id", "timestamp_location", "timestamp_location_dummy"),
                           by.y = c("device_id", "start_time", "end_time")) # this is the full dataset
  
  joined.data <- joined.data[order(joined.data$id, -joined.data$TrackDataIdVar), ]
  joined.data <- joined.data[!duplicated(joined.data$id),]
  joined.data <- joined.data[!is.na(TrackDataIdVar),,]
  joined.data[, c("timestamp_location_dummy", "TrackDataCreatedAt",  "i.timestamp"):=NULL]  
  locationslist[[i]] <- joined.data

}
saveRDS(locationslist, "output_files/locationslist.RDS")
```

We can also do that for all the tracks

``` r
all_unique_tracks_1101_0220 <- readRDS("output_files/all_unique_tracks_1101_0220.RDS")
locationslist2 <- list()
for(i in 1:length(folders)){
  locations<-fread(paste0(getwd(), "/Imput_data/",
                                       folders[i], 
                                       "/position_entries.csv"))
  setDT(locations)
  #locations <- locations[device_id%in%completed_atleast_one,,]
  locations[, c("heading", "speed", "created_at", "distance_between_previous_position", "desired_accuracy"):=NULL]
  
  locations[, timestamp_location:=as_datetime(timestamp, tz= "Europe/Paris")] # assign a real time stamp
  locations[, timestamp_location_dummy:=timestamp_location ]
  
  setkey(locations, device_id, timestamp_location, timestamp_location_dummy)
  setkey(all_unique_tracks_1101_0220, device_id, start_time, end_time)
  
  joined.data <- foverlaps(locations, all_unique_tracks_1101_0220, 
                           by.x = c("device_id", "timestamp_location", "timestamp_location_dummy"),
                           by.y = c("device_id", "start_time", "end_time")) # this is the full dataset
  
  joined.data <- joined.data[order(joined.data$id, -joined.data$TrackDataIdVar), ]
  joined.data <- joined.data[!duplicated(joined.data$id),]
  joined.data <- joined.data[!is.na(TrackDataIdVar),,]
  joined.data[, c("timestamp_location_dummy", "TrackDataCreatedAt",  "i.timestamp"):=NULL]  
  locationslist2[[i]] <- joined.data
print(i)
}
saveRDS(locationslist2, "output_files/locationslist_alltracks.RDS")
```

Due to memory (4GB) restrictions at statistics Netherlands it is not possible to `rbindlist(locationslist)` at once and we need to do this in smaller chunks.

``` r
rm(list=gdata::keep("locationslist"))
gc()
file1_5 <-rbind(locationslist[[1]],
                locationslist[[2]],
                locationslist[[3]],
                locationslist[[4]],
                locationslist[[5]])

saveRDS(file1_5, "output_files/temp_RDS_files/file1_5.RDS")
rm(file1_5)

file6_10 <-rbind(locationslist[[6]],
                 locationslist[[7]],
                 locationslist[[8]],
                 locationslist[[9]],
                 locationslist[[10]])
saveRDS(file6_10, "output_files/temp_RDS_files/file6_10.RDS")
rm(file6_10)

file11_15 <-rbind(locationslist[[11]],
                  locationslist[[12]],
                  locationslist[[13]],
                  locationslist[[14]],
                  locationslist[[15]])

saveRDS(file11_15, "output_files/temp_RDS_files/file11_15.RDS")
rm(file11_15)

file16_20 <-rbind(locationslist[[16]],
                  locationslist[[17]],
                  locationslist[[18]],
                  locationslist[[19]],
                  locationslist[[20]])

saveRDS(file16_20, "output_files/temp_RDS_files/file16_20.RDS")
rm(file16_20)

file21_25 <-rbind(locationslist[[21]],
                  locationslist[[22]],
                  locationslist[[23]],
                  locationslist[[24]],
                  locationslist[[25]])

saveRDS(file21_25, "output_files/temp_RDS_files/file21_25.RDS")
rm(file21_25)

file26_30<-rbind(locationslist[[26]],
                 locationslist[[27]],
                 locationslist[[28]],
                 locationslist[[29]],
                 locationslist[[30]])

saveRDS(file26_30, "output_files/temp_RDS_files/file26_30.RDS")
rm(file26_30)

file31_35<-rbind(locationslist[[31]],
                 locationslist[[32]],
                 locationslist[[33]],
                 locationslist[[34]],
                 locationslist[[35]])

saveRDS(file31_35, "output_files/temp_RDS_files/file31_35.RDS")
rm(file31_35)

file36_39<-rbind(locationslist[[36]],
                 locationslist[[37]],
                 locationslist[[38]],
                 locationslist[[39]])

saveRDS(file36_39, "output_files/temp_RDS_files/file36_39.RDS")
rm(file36_39)

rm(locationslist)


file1_5   <- readRDS("output_files/temp_RDS_files/file1_5.RDS")
file6_10  <- readRDS("output_files/temp_RDS_files/file6_10.RDS")
file11_15 <- readRDS("output_files/temp_RDS_files/file11_15.RDS")
file16_20 <- readRDS("output_files/temp_RDS_files/file16_20.RDS")

file1_20 <- rbind(file1_5,
                   file6_10,
                   file11_15,
                   file16_20)

saveRDS(file1_20, "output_files/temp_RDS_files/file1_20.RDS")

rm(file1_5)
rm(file6_10)
rm(file11_15)
rm(file16_20)
rm(file1_20)

file21_25 <- readRDS("output_files/temp_RDS_files/file21_25.RDS")
file26_30 <- readRDS("output_files/temp_RDS_files/file26_30.RDS")
file31_35 <- readRDS("output_files/temp_RDS_files/file31_35.RDS")
file36_39 <- readRDS("output_files/temp_RDS_files/file36_39.RDS")

file21_39 <- rbind(file21_25,
                  file26_30,
                  file31_35,
                  file36_39)
saveRDS(file21_39, "output_files/temp_RDS_files/file21_39.RDS")

rm(file21_25)
rm(file26_30)
rm(file31_35)
rm(file36_39)
rm(file21_39)
gc()


file1_20  <- readRDS("output_files/temp_RDS_files/file1_20.RDS")
file21_39 <- readRDS("output_files/temp_RDS_files/file26_30.RDS")
gc()

setDT(file1_20)
setDT(file21_39)

file1_39_2 <- rbind(file1_20,
                   file21_39)

rm(file1_20)
rm(file21_39)
saveRDS(file1_39_2, "output_files/temp_RDS_files/file1_39_2.RDS")
rm(file1_39_2)
locationswithtransport_2 <- readRDS("output_files/temp_RDS_files/file1_39_2.RDS")
#locationswithtransport_2 <- rbindlist(locationslist)
#locationswithtransport_full <- rbindlist(locationslist2)
saveRDS(locationswithtransport_2, "output_files/locationswithtransport_2.RDS")
saveRDS(locationswithtransport_full, "output_files/locationswithtransport_full.RDS")
```

Now we can save the individual tracks as files

``` r
locationswithtransport_2 <- readRDS("output_files/locationswithtransport_2.RDS")
locationswithtransport_full <- readRDS("output_files/locationswithtransport_full.RDS")
locationslist_alltracks.RDS <- readRDS("output_files/locationslist_alltracks.RDS")

setDT(locationswithtransport_full)
locationswithtransport_full_nomode <- locationswithtransport_full[is.na(mode3),,]                                    
locationswithtransport_2 %>%
  as_tibble() %>%
  summarise(n.device_ID = n_distinct(device_id),
            n.TrackDataIdVar= n_distinct(TrackDataIdVar),
            n.local_track_id = n_distinct(local_track_id))



locationswithtransport_2 %>%
  as_tibble() %>%
  group_by(device_id)%>%
  summarise(n.TrackDataIdVar_per_device_id= n_distinct(TrackDataIdVar))%>%
  arrange(desc(n.TrackDataIdVar_per_device_id))

locationswithtransport_full_nomode %>%
  as_tibble() %>%
  group_by(device_id)%>%
  summarise(n.TrackDataIdVar_per_device_id= n_distinct(TrackDataIdVar))%>%
  arrange(desc(n.TrackDataIdVar_per_device_id))

for(i in unique(locationswithtransport_2[, TrackDataIdVar])){
  fwrite(x = subset(locationswithtransport_2, TrackDataIdVar==i), 
         file= paste0("output_files/csv_per_track/csv_TrackDataIdVar_", i, ".csv"))
}
  
for(i in unique(locationswithtransport_2[, TrackDataIdVar])){
  saveRDS(object = subset(locationswithtransport_2, TrackDataIdVar==i), 
         file= paste0("output_files/RDS_per_track/RDS_TrackDataIdVar_", i, ".RDS"))
  print(i)
}


for(i in unique(locationswithtransport_full_nomode[, TrackDataIdVar])){
  saveRDS(object = subset(locationswithtransport_full, TrackDataIdVar==i), 
         file= paste0("output_files/RDS_per_track_unlabelled/RDS_TrackDataIdVar_", i, ".RDS"))
  print(i)
}
```

Or we can first merge the tracks that should be merged and then save them

``` r
locationswithtransport_2 <- readRDS("output_files/locationswithtransport_2.RDS")

setDT(locationswithtransport_2)


# one could use this code to just run the code on unique location entries
#locationswithtransport_2.unique <- unique(locationswithtransport_2, by= c("device_id",
#                                              "TrackDataIdVar", 
#                                              "start_time",  
#                                              "end_time",
#                                              "local_track_id",
#                                              "active_modes",
#                                              "mode",
#                                              "mode3", 
#                                              "latitude", 
#                                              "longitude", 
#                                              "accuracy", 
#                                              "altitude",  
#                                              "timestamp_location"))
#rm(locationswithtransport_2)

cut_off_merging_tracks <- 2 # in minutes
list_with_merged_tracks <-list()

for(i in unique(locationswithtransport_2[, device_id])){
  
  data <- subset(locationswithtransport_2, device_id==i)%>%
    mutate(TrackDataIdVar.copy = TrackDataIdVar) 
  
  test <- data %>%
    group_by(TrackDataIdVar.copy) %>%
    arrange(timestamp_location)%>%
    summarise(start = min(timestamp_location),
              end   = max(timestamp_location),
              mode3 = mode3[1]) %>%
    arrange(start)%>%
    mutate(start.lead= dplyr::lead(start, n = 1, default = NA)) %>%
    mutate(mode3.lead = dplyr::lead(mode3, n = 1, default = NA_character_)) %>%
    mutate(diff_end_start.lead = start.lead- end ) %>%
    mutate(merge = ifelse(mode3 == mode3.lead & start.lead- end < cut_off_merging_tracks, TRUE, FALSE)) %>%
    mutate(merge = ifelse(is.na(merge), FALSE, merge)) %>%
    mutate(TrackDataIdVar.copy.lead = dplyr::lead(TrackDataIdVar.copy, n = 1, default = NA)) %>%
    select(TrackDataIdVar.copy, TrackDataIdVar.copy.lead,  merge, mode3)
  
  while(sum(test$merge, na.rm = T)> 0){
    test <- data %>%
      group_by(TrackDataIdVar.copy) %>%
      arrange(timestamp_location)%>%
      summarise(start = min(timestamp_location),
                end   = max(timestamp_location),
                mode3 = mode3[1]) %>%
      arrange(start)%>%
      mutate(start.lead= dplyr::lead(start, n = 1, default = NA)) %>%
      mutate(mode3.lead = dplyr::lead(mode3, n = 1, default = NA_character_)) %>%
      mutate(diff_end_start.lead = start.lead- end ) %>%
      mutate(merge = ifelse(mode3 == mode3.lead & start.lead- end < cut_off_merging_tracks, TRUE, FALSE)) %>%
      mutate(merge = ifelse(is.na(merge), FALSE, merge))%>%
      mutate(TrackDataIdVar.copy.lead = dplyr::lead(TrackDataIdVar.copy, n = 1, default = NA)) %>%
      select(TrackDataIdVar.copy, TrackDataIdVar.copy.lead,  merge, mode3)
    
    
    data <- test %>%
      left_join(data,  by = c("TrackDataIdVar.copy", "mode3"))%>%
      as_tibble() %>%
      mutate(TrackDataIdVar.copy = if_else(merge == TRUE, TrackDataIdVar.copy.lead, TrackDataIdVar.copy)) %>%
      select(-TrackDataIdVar.copy.lead, -merge)
    
  }
  
  list_with_merged_tracks[[i]] <- data
  
}
new_locationswithtransport_2 <- do.call(rbind, list_with_merged_tracks)

setDT(new_locationswithtransport_2)
new_locationswithtransport_2 %>%
  group_by(TrackDataIdVar.copy)%>%
  summarise(n_merged = n_distinct(TrackDataIdVar)) %>%
  arrange(desc(n_merged))%>%
  filter(n_merged>1)

new_locationswithtransport_2 %>%
  as_tibble()%>%
  filter(is.na(TrackDataIdVar.copy))%>%
  select(device_id,TrackDataIdVar, timestamp_location,TrackDataIdVar.copy)


for(i in unique(new_locationswithtransport_2[, TrackDataIdVar.copy])){
  saveRDS(object = subset(new_locationswithtransport_2, TrackDataIdVar.copy==i), 
          file= paste0("output_files/RDS_per_track_merged_tracks/RDS_TrackDataIdVar.copy_", i, ".RDS"))
    print(i)

}
```

Functions
---------

### Function to clean the data

This is the function used to clean and smooth the data. Later on this function is called on for all the individual trips. `des.acc` and `des.max.speed` can respectively be used to set the max accuracy and max speed (between adjacent points) filters. The other inputs serve as the different parameters of the smoothing algorithms.

``` r
GPSdatacleaner <- function(GPSdataframe,
                           do_filter_speed_accuracy=TRUE,
                           station.data   = station.data,
                           des.acc        = 200, 
                           des.max.speed  = 150, 
                           long_time_diff = 300, 
                           impute         = TRUE, 
                           impute.NS      = FALSE,
                           distance.hitbox.trainstation = .2,
                           Savitzky.Golay = FALSE,
                           Savitzky.Golay.p = 3, 
                           Savitzky.Golay.n = 101,
                           Kalman.filter.accuracy = FALSE,
                           interpolate = TRUE,
                           R_mult.Kalman_filter_accuracy = 2,
                           Q_mult.Kalman_filter_accuracy = 1.5,
                           Kalman.filter.smooth = FALSE,
                           R_value_smooth_Kalman = 100
) 

{
  # names of longitude and latitude should be exact that
  # calculate the distance between to points
  #The shortest distance between two points (i.e., the 'great-circle-distance' or 'as the crow flies'), according to the 'Vincenty (sphere)'
  #method. This method assumes a spherical earth, ignoring ellipsoidal effects and it is less accurate then the distVicentyEllipsoid method.
  # https://stackoverflow.com/questions/27928/calculate-distance-between-two-latitude-longitude-points-haversine-formula
  
  # add whether a value was imputed or not.
  
#as.numeric(GPSdataframe[249, ]$timestamp_location) == as.numeric(GPSdataframe[250, ]$timestamp_location)


  GPSdataframe <- GPSdataframe %>%
    mutate(latitude_imputed  = latitude)          %>%
    mutate(longitude_imputed  = longitude)       
  
  if(!is.null(station.data)){
    
    countDataPoints2         <- function(p) {
      distances              <- spDistsN1(data.matrix(GPSdataframe[,c("longitude","latitude")]), 
                                          p,
                                          longlat=TRUE) # in km
      return(which(distances <= distance.hitbox.trainstation))
    }
    GPSdataframe$rowID                <- 1:nrow(GPSdataframe)
    datapoints.by.trainstation        <- apply(data.matrix(station.data[,c("Lon","Lat")]), 1, countDataPoints2)
    if(length(datapoints.by.trainstation) !=length(station.data$Namen.Kort)){ ## GET THIS TO WORK
      GPSdataframe <- GPSdataframe %>%
        mutate(trainstation.nearby = FALSE,
               L1 = NA_character_,
               near.train.station = NA)
    }else{
      names(datapoints.by.trainstation) <- station.data$Namen.Kort
      long.data.frame                   <- melt(datapoints.by.trainstation)
      GPSdataframe$near.train.station   <- NA
      GPSdataframe                      <- GPSdataframe %>%
        left_join(long.data.frame, by   = c("rowID" = "value")) %>%
        mutate(trainstation.nearby      = if_else(is.na(L1), F, T ))
    }
    if(impute.NS == TRUE){
      GPSdataframe <- left_join(GPSdataframe, station.data, by = c("L1"= "Namen.Kort")) %>%
        mutate(latitude_imputed  = if_else(trainstation.nearby == TRUE, Lat,latitude))          %>%
        mutate(longitude_imputed  = if_else(trainstation.nearby == TRUE, Lon,longitude))         %>%
        mutate(filter2   = if_else(trainstation.nearby == TRUE, TRUE, FALSE))    }
    
    
  }else{
    GPSdataframe <- GPSdataframe 
  }
  
  if(do_filter_speed_accuracy==TRUE){
    GPSdataframe2 <- GPSdataframe %>%
      group_by(device_id) %>%
      arrange(device_id,timestamp_location) %>%
      mutate(lag.latitude_imputed  = dplyr::lag(latitude_imputed, n = 1, default = NA))  %>%
      mutate(lag.longitude_imputed = dplyr::lag(longitude_imputed, n = 1, default = NA)) %>%
      mutate(lag.longitude_imputed = if_else(is.na(lag.longitude_imputed), longitude_imputed, lag.longitude_imputed)) %>%
      mutate(lag.latitude_imputed  = if_else(is.na(lag.latitude_imputed), latitude_imputed, lag.latitude_imputed))    %>%
      mutate(distance.between.points = distVincentySphere(matrix(c(longitude_imputed, latitude_imputed), ncol=2), 
                                                          matrix(c(lag.longitude_imputed, lag.latitude_imputed), ncol=2))) %>% # in meters
      mutate(bearing        = geosphere::bearing(matrix(c(longitude_imputed, latitude_imputed), ncol=2), 
                                                 matrix(c(lag.longitude_imputed, lag.latitude_imputed), ncol=2))) %>%
      mutate(abs.bearing    = if_else(bearing == -180, 0, abs(bearing)))          %>%
      mutate(rel_bearing    = abs(abs.bearing - dplyr::lag(abs.bearing, n = 1, default = NA))) %>% # is the relative bearing compared to to previous point 
      mutate(time_diff      = c(0, diff(timestamp_location,1))) %>%
      mutate(time_diff_long = if_else(time_diff > long_time_diff, T, F)) %>%
      mutate(segment        = as.factor(cumsum(time_diff_long)))         %>%   # devide the trip in different segments
      mutate(segment_id     = paste0(device_id, "_", segment))           %>% 
      mutate(speed          = distance.between.points / time_diff)       %>%
      mutate(speed          = if_else(is.nan(speed), 0, speed))          %>%
      mutate(speed          = if_else(speed == Inf, 10000, 3.6*speed))   %>% # to get to km/h
      mutate(longitude_imputed      = replace(longitude_imputed, accuracy > des.acc | speed > des.max.speed , NA)) %>%
      mutate(latitude_imputed       = replace(latitude_imputed, accuracy > des.acc | speed > des.max.speed , NA))  %>%
      mutate(distance.between.points = replace(distance.between.points, accuracy > des.acc | speed > des.max.speed , NA)) %>%
      mutate(speed                   = replace(speed, accuracy > des.acc | speed > des.max.speed , NA)) %>%
      mutate(filter1                 = if_else(is.na(longitude_imputed), T, F))  %>%
      mutate(speed.mean5             = rollapply(speed,
                                                 width= list(c(-5, -4, -3, -2, -1)),
                                                 FUN = mean,
                                                 na.rm= T,
                                                 fill = NA))   %>% 
      mutate(acceleration   = c(0, diff(speed,1)) / time_diff) %>%
      arrange(device_id, timestamp_location)
    
    # filter1 = filter because too high speed or too low accuracy
  }else{
    GPSdataframe2 <- GPSdataframe %>%
      group_by(device_id) %>%
      arrange(device_id,timestamp_location) %>%
      mutate(lag.latitude_imputed  = dplyr::lag(latitude_imputed, n = 1, default = NA))  %>%
      mutate(lag.longitude_imputed = dplyr::lag(longitude_imputed, n = 1, default = NA)) %>%
      mutate(lag.longitude_imputed = if_else(is.na(lag.longitude_imputed), longitude_imputed, lag.longitude_imputed)) %>%
      mutate(lag.latitude_imputed  = if_else(is.na(lag.latitude_imputed), latitude_imputed, lag.latitude_imputed))    %>%
      mutate(distance.between.points = distVincentySphere(matrix(c(longitude_imputed, latitude_imputed), ncol=2), 
                                                          matrix(c(lag.longitude_imputed, lag.latitude_imputed), ncol=2))) %>% # in meters
      mutate(bearing        = geosphere::bearing(matrix(c(longitude_imputed, latitude_imputed), ncol=2), 
                                                 matrix(c(lag.longitude_imputed, lag.latitude_imputed), ncol=2))) %>%
      mutate(abs.bearing    = if_else(bearing == -180, 0, abs(bearing)))          %>%
      mutate(rel_bearing    = abs(abs.bearing - dplyr::lag(abs.bearing, n = 1, default = NA))) %>% # is the relative bearing compared to to previous point 
      mutate(time_diff      = c(0, diff(timestamp_location,1))) %>%
      mutate(time_diff_long = if_else(time_diff > long_time_diff, T, F)) %>%
      mutate(segment        = as.factor(cumsum(time_diff_long)))         %>%   # devide the trip in different segments
      mutate(segment_id     = paste0(device_id, "_", segment))           %>% 
      mutate(speed          = distance.between.points / time_diff)       %>%
      mutate(speed          = if_else(is.nan(speed), 0, speed))          %>%
      mutate(speed          = if_else(speed == Inf, 10000, 3.6*speed))   %>% # to get to km/h
      mutate(filter1                 = if_else(is.na(longitude_imputed), T, F))  %>%
      mutate(speed.mean5             = rollapply(speed,
                                                 width= list(c(-5, -4, -3, -2, -1)),
                                                 FUN = mean,
                                                 na.rm= T,
                                                 fill = NA))   %>% 
      mutate(acceleration   = c(0, diff(speed,1)) / time_diff) %>%
      arrange(device_id, timestamp_location)
  }
  
  if(impute == T){
    
        GPSdataframe2_list <- list()
    
    for(i in unique(GPSdataframe2$segment_id)){
      GPSdataframe2.1 <- GPSdataframe2 %>%
        filter(segment_id== i) 
      # some files are so small we need this filter to make sure the code does not crash when having only 1 GPS coordinate in a track/
      # no real filtering can happen if there is only one coordinate though.
      # if all acuuacry values are above the threshold (and thus deleted, we also have to use the original values)
      if(nrow(GPSdataframe2.1)==1 | all(GPSdataframe2.1$accuracy > des.acc) |  all(is.na(GPSdataframe2.1$latitude_imputed))){
        GPSdataframe2.2 <- GPSdataframe2.1      %>%
          group_by(device_id, segment_id) %>%
          arrange(timestamp_location) %>%
          mutate(latitude_imputedT  = latitude) %>%
          mutate(longitude_imputedT = longitude)    %>%
          arrange(device_id,timestamp_location) 
        
      }else{holder_lon_lat<- GPSdataframe2.1      %>%
        group_by(device_id, segment_id) %>%
        arrange(timestamp_location) %>%
        mutate(latitude_imputedT  = if_else(time_diff_long == T, 
                                            if_else(is.na(lag.latitude_imputed),  
                                                    zoo::na.locf(lag.latitude_imputed,  fromLast = T), 
                                                    lag.latitude_imputed),  
                                            latitude_imputed)) %>%
        mutate(longitude_imputedT = if_else(time_diff_long == T, 
                                            if_else(is.na(lag.longitude_imputed), 
                                                    zoo::na.locf(lag.longitude_imputed, fromLast = T), 
                                                    lag.longitude_imputed), 
                                            longitude_imputed)) %>%
        dplyr::select(device_id, segment_id, timestamp_location, longitude_imputedT, latitude_imputedT)
      
      longitude_imputedTvec <- imputeTS::na.locf(na.approx(holder_lon_lat$longitude_imputedT,na.rm = FALSE), option = "nocb")
      latitude_imputedTvec <- imputeTS::na.locf(na.approx(holder_lon_lat$latitude_imputedT,na.rm = FALSE), option = "nocb")
      
      GPSdataframe2.2 <- GPSdataframe2.1%>%
        group_by(device_id, segment_id) %>%
        arrange(timestamp_location) %>%
        mutate(latitude_imputedT  = latitude_imputedTvec) %>%
        mutate(longitude_imputedT = longitude_imputedTvec) 
      
      }
      GPSdataframe2_list[[i]] <- GPSdataframe2.2
      
    }
    GPSdataframe2 <- rbindlist(GPSdataframe2_list)
    GPSdataframe3 <- GPSdataframe2%>%
      arrange(timestamp_location) %>%
      group_by(device_id, segment_id) %>%
      arrange(timestamp_location)
    
    
    
    
  }else {
    GPSdataframe3 <- GPSdataframe2 %>%
      mutate(latitude_imputedT  = latitude_imputed) %>%
      mutate(longitude_imputedT =  longitude_imputed)  
  }
  
  if(interpolate == TRUE){
    
    GPSdataframe4 <- GPSdataframe3 %>%
      mutate(timestamp_location_copy = as.character(timestamp_location))%>% # this is bit of an ugly solution to the problem (to use character instead of exact time stamp, but for now it works)
      # roll = "nearest" does not give satisfying results.
      as_tibble() 
    
    start <- min(GPSdataframe4$timestamp_location)
    end <- max(GPSdataframe4$timestamp_location)
    dat <- data.frame(time_forced= seq(start, end, 1), time_forced_copy =  seq(start, end, 1))%>%
      mutate(time_forced = as.character(time_forced))%>%
      as_tibble()
    
    #GPSdataframe4<- left_join(dat, GPSdataframe4, by= c("time_forced"= "timestamp_location_copy"))
    
    
    setDT(GPSdataframe4)
    setDT(dat)
    setkey(dat,time_forced)
    setkey(GPSdataframe4, timestamp_location_copy)
    
    GPSdataframe4 <- GPSdataframe4[dat, roll= FALSE]
    GPSdataframe4[, interpolated := FALSE][is.na(local_track_id), interpolated := TRUE]
    
    GPSdataframe4 <- GPSdataframe4%>%
      arrange(timestamp_location_copy)%>%
      mutate(latitude_imputedT  = zoo::na.locf(na.approx(latitude_imputedT, na.rm = FALSE),  fromlast = TRUE))  %>% # just use just a  rolling imputation of a straight line between two points
      mutate(longitude_imputedT = zoo::na.locf(na.approx(longitude_imputedT, na.rm = FALSE),  fromlast = TRUE)) %>% 
      mutate(accuracy = zoo::na.locf(na.approx(accuracy, na.rm = FALSE),  fromlast = TRUE)) %>% # this interpolation is later needed for one of the Kalman Filters
      mutate(segment_id =  zoo::na.locf(segment_id)) %>% # this interpolation is later needed for one of the Kalman Filters
      mutate(mode3= zoo::na.locf(mode3, na.rm = FALSE))%>%
      mutate(local_track_id= zoo::na.locf(local_track_id, na.rm = FALSE)) %>%
      mutate(device_id= zoo::na.locf(device_id, na.rm = FALSE))
  }else{
    GPSdataframe4 <- GPSdataframe3 %>%
      as_tibble() %>%
      mutate(timestamp_location_copy = as.character(timestamp_location))%>%
      mutate(time_forced = as.character(timestamp_location))%>%
      mutate(time_forced_copy = timestamp_location)
  }
  i
  if(Savitzky.Golay == TRUE){
    dataframe_smoothed_segments <- list()
    for(i in unique(GPSdataframe4$segment_id)){
      GPSdataframe.raw        <- GPSdataframe4[GPSdataframe4$segment_id == i, ]
      Savitzky.Golay.smoothed.step1 <- GPSdataframe4[GPSdataframe4$segment_id == i, ]
      if(nrow(Savitzky.Golay.smoothed.step1)>Savitzky.Golay.n){
        p <- Savitzky.Golay.p
        n <- Savitzky.Golay.n
      } else{  # for very short segments it is important to overrule user settings, too make sure the smoothing algorithm can be fit. 
        p <- 3
        n <- p + 3 - p%%2
      }
      
      Savitzky.Golay.smoothed.step2 <- Savitzky.Golay.smoothed.step1 %>%
        mutate(short_time = as.numeric(time_forced_copy - min(Savitzky.Golay.smoothed.step1$time_forced_copy)))%>% 
        ungroup()          %>%
        arrange(time_forced_copy) %>%
        dplyr::select(timestamp_location_copy, longitude_imputedT,latitude_imputedT, short_time, time_forced_copy)
      
      
      Savitzky.Golay.smoothed.step3<-TrajFromCoords(Savitzky.Golay.smoothed.step2,xCol = "longitude_imputedT", yCol = "latitude_imputedT",  timeCol = "short_time" )
      
      if(nrow(Savitzky.Golay.smoothed.step3)==1){
        Savitzky.Golay.smoothed.angle <-0
      }else{
        Savitzky.Golay.smoothed.angle <- c(0, TrajAngles(Savitzky.Golay.smoothed.step3), 0) # first and last angle are 0.
      }
      if(all(Savitzky.Golay.smoothed.step3$displacement=="0+0i")){
        Savitzky.Golay.smoothed.step4 <- Savitzky.Golay.smoothed.step3%>%
          dplyr::select(y, x, time_forced_copy) %>%
          bind_cols(Savitzky.Golay.smoothed.angle = Savitzky.Golay.smoothed.angle)
      }else{
        Savitzky.Golay.smoothed.step4 <-  Savitzky.Golay.smoothed.step3 %>%
          TrajSmoothSG(p=p, n=n) %>%
          dplyr::select(y, x, time_forced_copy) %>%
          bind_cols(Savitzky.Golay.smoothed.angle = Savitzky.Golay.smoothed.angle )
      }
      
      
      setDT(Savitzky.Golay.smoothed.step4)
      Savitzky.Golay.smoothed.step4 <-  unique(Savitzky.Golay.smoothed.step4, by ="time_forced_copy")
      dataframe_smoothed_segments[[i]] <- left_join(GPSdataframe.raw, Savitzky.Golay.smoothed.step4, by= "time_forced_copy")
    }
    
    
    GPSdataframe5 <- do.call(rbind, dataframe_smoothed_segments) %>%
      arrange(timestamp_location)%>%
      dplyr::select(-latitude_imputedT, -longitude_imputedT)%>%
      rename(latitude_imputedT =y,
             longitude_imputedT = x,
             Savitzky.Golay.smoothed.angle.full = Savitzky.Golay.smoothed.angle )
    
    

  }else {
    GPSdataframe5 <- GPSdataframe4 
  }
  
  if(Kalman.filter.accuracy == TRUE){
    Kalman_filter_accuracy <- function(data = GPSdataframe5,  
                                       R_mult= R_mult.Kalman_filter_accuracy, 
                                       Q_mult=  Q_mult.Kalman_filter_accuracy){
      
      #create a time variable 
      setDT(data)
      data[, short_time := time_forced_copy - min(data$time_forced_copy)]
      data[, id := .I - 1]
      
      #initializing variables
      count <- nrow(data) # amount of data points in the data
      z <- cbind(data$longitude_imputedT, data$latitude_imputedT) #measurements
      
      #Allocate space:
      xhat <- matrix(rep(0, 2 * count), ncol = 2) #a posteri estimate at each step
      P <- array(0, dim = c(2, 2, count))  #a posteri error estimate
      xhatminus <- matrix(rep(0, 2 * count), ncol = 2) #a priori estimate
      Pminus <- array(0, dim = c(2, 2, count)) #a priori error estimate
      K <- array(0, dim = c(2, 2, count)) #gain
      
      #Initializing matrices
      A <- diag(2)
      H <- diag(2)
      R <- function(k) diag(2) * data$accuracy[k]^R_mult #estimate of measurement variance
      Q <- function(k) diag(2) * as.numeric(data$short_time[k - 1])^Q_mult # the process variance
      
      #initialise guesses:
      xhat[1, ] <- z[1, ]
      P[, , 1] <- diag(2)
      k <- 2
      for (k in 2:count) {
        #time update
        #project state ahead
        xhatminus[k, ] <- A %*% xhat[k - 1, ] #+ B %*% u[k-1]
        
        #project error covariance ahead
        Pminus[, , k] <- A %*% P[, , k - 1] %*%  t(A) + (Q(k))
        
        #measurement update
        # kalman gain
        K[, , k] <- Pminus[, , k] %*% t(H) / (H %*% Pminus[, , k] %*% t(H) + R(k))
        
        #what if NaaN?
        K[, , k][which(is.nan(K[, , k]))] <- 0
        
        # update estimate with measurement
        xhat[k, ] <-  xhatminus[k, ] + K[, , k] %*% (z[k, ] - H %*% xhatminus[k, ])
        #update error covariance
        P[, , k] = (diag(2) - K[, , k] %*% H) %*% Pminus[, , k]
      }
      
      xhat <- setDT(data.frame(xhat))
      names(xhat) <- c("longitude_imputedT", "latitude_imputedT")
      xhat$Q_mult <- Q_mult
      xhat$R_mult <- R_mult
      xhat
    }
    
    GPSdataframe6<- bind_cols(dplyr::select(GPSdataframe5, 
                                            -latitude_imputedT, 
                                            -longitude_imputedT),
                              Kalman_filter_accuracy(data= GPSdataframe5,
                                                     R_mult= R_mult.Kalman_filter_accuracy,
                                                     Q_mult=  Q_mult.Kalman_filter_accuracy))
  } 
  else{
    GPSdataframe6 <- GPSdataframe5 
  }  
  
  if(Kalman.filter.smooth==TRUE){
    pos2<-matrix(c(GPSdataframe6$longitude_imputedT,GPSdataframe6$latitude_imputedT), ncol=2)
    
    Kalman_filter_smoothing <- function(pos, R_value=R_value_smooth_Kalman) {
      
      kalmanfilter <- function(z) {
        ## predicted state and covariance
        xprd <- A %*% xest
        pprd <- A %*% pest %*% t(A) + Q
        
        ## estimation
        S <- H %*% t(pprd) %*% t(H) + R
        B <- H %*% t(pprd)
        
        kalmangain <- t(solve(S, B))
        
        ## estimated state and covariance
        ## assigned to vars in parent env
        xest <<- xprd + kalmangain %*% (z - H %*% xprd)
        pest <<- pprd - kalmangain %*% H %*% pprd
        
        ## compute the estimated measurements
        y <- H %*% xest
      }
      
      dt <- 1
      A <- matrix( c( 1, 0, dt, 0, 0, 0,  # x
                      0, 1, 0, dt, 0, 0,   # y
                      0, 0, 1, 0, dt, 0,   # Vx
                      0, 0, 0, 1, 0, dt,   # Vy
                      0, 0, 0, 0, 1,  0,   # Ax
                      0, 0, 0, 0, 0,  1),  # Ay
                   6, 6, byrow=TRUE)
      H <- matrix( c(1, 0, 0, 0, 0, 0,
                     0, 1, 0, 0, 0, 0),
                   2, 6, byrow=TRUE)
      Q <- diag(6)
      R <- R_value * diag(2)
      N <- nrow(pos)
      Y <- matrix(NA, N, 2)
      
      #xest <- matrix(0, 6, 1)
      xest <- matrix(c(pos[1,], 0,0,0,0),6,1)
      pest <- matrix(0, 6, 6)
      
      for (i in 1:N) {
        Y[i,] <- kalmanfilter(t(pos[i,,drop=FALSE]))
      }
      Y <- as.data.frame(Y)
      Y <- Y %>%
        rename(longitude_imputedT= V1,
               latitude_imputedT = V2)
    }
    GPSdataframe7 <- bind_cols(dplyr::select(GPSdataframe6, 
                                             -latitude_imputedT, 
                                             -longitude_imputedT), Kalman_filter_smoothing(pos2, R_value = R_value_smooth_Kalman))
  }else{
    GPSdataframe7 <- GPSdataframe6 
  }
  
  GPSdataframe.final <-GPSdataframe7 
  
  return(GPSdataframe.final)
  
}


# impute data tussen twee punten als er een gat groter is dan 10 minuten.
# link a google maps of osm, of OV. 
# filteren op datapunten met precies dezelfde tijd (op de milliseconde)
#https://stackoverflow.com/questions/50694746/calculating-distance-between-two-points-using-the-distm-function-inside-mutate

#filter is based on:
# https://www2.stetson.edu/mathcs/wp-content/uploads/2015/10/jmichaels-final.pdf
# https://cran.rstudio.com/web/packages/trajr/vignettes/trajr-vignette.html
# https://www.star.bnl.gov/~gorbunov/main/node27.html
```

This is an example of how the cleaner function works. for some reason a lot of data was registered twice in the GPS data (same lat/lon and exact same time stamp, but different id), so these need to be filtered out.

``` r
file_26115 <- readRDS("output_files/RDS_per_track/RDS_TrackDataIdVar_26115.RDS")
file_26115.unique <- unique(file_26115, by= c("device_id",
                         "TrackDataIdVar", 
                         "start_time",  
                         "end_time",
                         "local_track_id",
                         "mode3", 
                         "latitude", 
                         "longitude", 
                         "accuracy", 
                         "altitude",  
                         "timestamp_location"))


station.data <- fread("external_imput_data/trainstations.csv")
file_26115.unique_filtered <- GPSdatacleaner(file_26115.unique, station.data = station.data,  interpolate = TRUE)

ggplot()+
  geom_point(data = file_26115.unique_filtered, aes(x=longitude_imputedT, y= latitude_imputedT, col= interpolated), size=2)+
  geom_point(data= filter(file_26115.unique_filtered, filter1 == T),aes(x=longitude, y= latitude), col= "red", size= 4, alpha=.4)
```

### Functions to check whether location features are nearby

This function checks whether or not a location feature is nearby, but does not check if there are multiple of the same feature nearby and it does also not save the names of the nearby features

``` r
# this app could be much faster if it would filter by duplicate GPS coordinates

check_if_close_fast <- function(dataset1,       
                            dataset2,     
                            desired.dist = .2,
                            margin       = 0.0025,
                            margin2      = 0.01){
  
  # dataset1 needs at least the columns 
  #  - "id_for_filter", 
  #  - "device_id"
  #  - "latitude"
  #  - "longitude"
  
  # dataset2 needs at least the columns 
  #  - "id", 
  #  - "name"
  #  - "latitude"
  #  - "longitude"
  
  # the margin should not be too small, because that will lead to the missing of matches. 
  

  # 1) we only select the variables we need from dataset 1
  dataset1_orig <- copy(dataset1)
  #dataset2_orig <- copy(dataset2)
  
  dataset1 <- setDT(dataset1)[,c("id_for_filter", "device_id", "latitude", "longitude")]
  setnames(dataset1, old = c("id_for_filter", "latitude", "longitude"), new = c("id_dataset1", "latitude_gps", "longitude_gps"))
  
  # 2) we only select the variables we need from dataset 2
  dataset2 <- setDT(dataset2)[,c("id", "name", "latitude", "longitude")]
  setnames(dataset2, old = c("id", "latitude", "longitude"), new = c("id_dataset2", "latitude_feature", "longitude_feature"))
  
  # 3) only keep subet of dataset2 that falls within dataset 1. 
  #    There is no reason to check if features are close that already fall out of the GPS coordinates in the trip we want to check
  #    We do add a small point margin around it to be on the save side. Maybe a feature falls just out the GPS coordinates, 
  #    but is still near to a GPS point
  dataset2 <- dataset2[latitude_feature  %between%  (range(dataset1$latitude_gps) + c(-margin2, +margin2)) 
                       & longitude_feature %between% (range(dataset1$longitude_gps) + c(-margin2, +margin2)), ]
  
  if(dim(dataset2)[1]==0){

    setDT(dataset1_orig)
    finallist <- dataset1_orig[, nearby := FALSE][, .(nearby=nearby, device_id=device_id, id_for_filter=id_for_filter),]
    
  }else{
  # 4) we copy the original coordinates and add the margin for longitude and latitude
  setkey(dataset1, latitude_gps)
  
  dataset1[, gps_lat := latitude_gps + 0]
  dataset1[, gps_lon := longitude_gps + 0]

  
  dataset2[, lat := latitude_feature  + 0]
  dataset2[, lon := longitude_feature + 0]
  

  setkey(dataset1, latitude_gps, gps_lat)
  setkey(dataset2, lat)
  
  #dataset1_by_lat <- dataset1[, .(id), keyby = "gps_lat"]
  
  
  By_latitude <- 
    dataset2[dataset1, 
                 on = "lat==latitude_gps",
                 
                 # within margin of latitude
                 roll = margin, 
                 # +/-
                 rollends = c(TRUE, TRUE),
                 
                 # and remove those beyond 0.5 degrees
                 nomatch=0L] %>%
    .[, .(id_lat = id_dataset1,
          name = name,
          latitude_feature = latitude_feature,
          longitude_feature = longitude_feature,
          gps_lat,
          gps_lon),
      keyby = .(lon = gps_lon)]
  
  setkey(dataset2, lon)
  
  By_latlon <-
    dataset2[By_latitude,
                 on = c("name", "lon"),
                 
                 # within margin of longitude
                 roll = margin, 
                 # +/-
                 rollends = c(TRUE, TRUE),
                 # and remove those beyond 0.5 degrees
                 nomatch=0L]
  
  By_latlon[, distance := haversine_distance(lat1 = gps_lat, 
                                             lon1 = gps_lon,
                                             lat2 = latitude_feature,
                                             lon2 = longitude_feature)]
  
  valuesnearstop<-By_latlon[, nearby := if_else(distance<desired.dist, TRUE, FALSE)]
  

  
  setkey(valuesnearstop, id_lat)
  setDT(dataset1_orig)
  setkey(dataset1_orig, id_for_filter)
  
  
  finallist <- valuesnearstop[dataset1_orig][, .(nearby=nearby, distance= distance, device_id=device_id, id_for_filter=id_lat),]
  
  }
  return(finallist)
}
```

This function checks whether or not a a context feature is nearby, but does so in a slightly slower manner. However, it will save the names of the features it is close to and will check whether multiple features are within the desired distance.

``` r
check_if_close2 <- function(dataset1,       
                            dataset2,       
                            desired.dist = .2,
                            margin       = 0.005,
                            margin2      = 0.1){
  
  
  
  # dataset1 needs at least the columns 
  #  - "id_for_filter", 
  #  - "device_id"
  #  - "latitude"
  #  - "longitude"
  
  # dataset2 needs at least the columns 
  #  - "id", 
  #  - "name"
  #  - "latitude"
  #  - "longitude"
  
  # the margin should not be too small, because that will lead to the missing of matches. 
  
  
  
  # 1) we only select the variables we need from dataset 1
  dataset1_orig <- copy(dataset1)
  
  dataset1 <- setDT(dataset1)[,c("id_for_filter", "device_id", "latitude", "longitude")]
  setnames(dataset1, old = c("id_for_filter", "latitude", "longitude"), new = c("id_dataset1", "latitude_gps", "longitude_gps"))
  
  # 2) we only select the variables we need from dataset 2
  dataset2 <- setDT(dataset2)[,c("id", "name", "latitude", "longitude")]
  setnames(dataset2, old = c("id", "latitude", "longitude"), new = c("id_dataset2", "latitude_feature", "longitude_feature"))
  
  # 3) only keep subet of dataset2 that falls within dataset 1. 
  #    There is no reason to check if features are close that already fall out of the GPS coordinates in the trip we want to check
  #    We do add a 0.01 point margin around it to be on the save side. Maybe a feature falls just out the GPS coordinates, 
  #    but is still near to a GPS point
  dataset2 <- dataset2[latitude_feature  %between%  (range(dataset1$latitude_gps) + c(-margin2, +margin2)) 
                       & longitude_feature %between% (range(dataset1$longitude_gps) + c(-margin2, +margin2)), ]
  
  
  # 4) we copy the original coordinates and add the margin for longitude and latitude
  setkey(dataset1, latitude_gps)
  
  dataset1[, gps_lat := latitude_gps + 0]
  dataset1[, gps_lon := longitude_gps + 0]
  
  dataset2[, lat := latitude_feature  + 0]
  dataset2[, lon := longitude_feature + 0]
  
  dataset2[, feature.lat.minus := latitude_feature  - margin]
  dataset2[, feature.lat.plus  := latitude_feature  + margin]
  
  setkey(dataset1, latitude_gps, gps_lat)
  setkey(dataset2, feature.lat.minus, feature.lat.plus)
  
  
  
  
  check<-foverlaps(dataset1, dataset2, nomatch = 0L, type="within")[, .(longitude_feature=longitude_feature,
                                                                        latitude_feature=latitude_feature,
                                                                        name=name,
                                                                        gps_lat=latitude_gps,
                                                                        gps_lon=longitude_gps,
                                                                        gps_lon2=longitude_gps,
                                                                        id_for_filter=id_dataset1,
                                                                        device_id=device_id)]

  
  dataset2[, longitude_feature_minus:= longitude_feature - margin]
  dataset2[, longitude_feature_plus := longitude_feature  + margin]
  
  
  setDT(check)
  setkey(check, gps_lon, gps_lon2)
  setkey(dataset2, longitude_feature_minus, longitude_feature_plus)
  
  check2 <- foverlaps(check, dataset2, nomatch = 0L, type="within")[, .(longitude_feature=longitude_feature,
                                                                        latitude_feature=latitude_feature,
                                                                        name=name,
                                                                        gps_lat=gps_lat,
                                                                        gps_lon=gps_lon,
                                                                        id_for_filter=id_for_filter,
                                                                        device_id=device_id)]
  
  
  # 5) this is the actual function we use to check if a datapoint is nearby, by using the haversine method
  
  valuesnearstop <- check2[, distance := haversine_distance(lat1 = gps_lat, 
                                                            lon1 = gps_lon,
                                                            lat2 = latitude_feature,
                                                            lon2 = longitude_feature)]
  valuesnearstop[, nearby := if_else(distance<desired.dist, TRUE, FALSE)]
  
  # 6) now we left join the original dataset and and the data point that are near a feature
  # 7) we add a new logical variable to check if any bus stop is near
  setDT(dataset1_orig)
  setDT(valuesnearstop)
  
  setkey(dataset1_orig, "id_for_filter")
  setkey(valuesnearstop, "id_for_filter")
  
  # add a dummy to check if any bus stop is nearby.
  
  fulldata <- valuesnearstop %>%
    mutate(name = if_else(nearby == TRUE, name, NA_character_))
  
  #fulldata <- valuesnearstop[dataset1_orig][, nearby := TRUE][is.na(name), nearby := FALSE]
  
  
  # 8) if a point is near multiple features at once these are listed in a list,
  #     instead of having duplicate rows with the same id but different features
  finallist <- unique(setDT(fulldata)[order(id_for_filter, name), list(feature_name=list(name), nearby=nearby, distance), by="id_for_filter"], by="id_for_filter")
  #finallist <- unique(setDT(fulldata)[order(id_for_filter,distance, name), list(feature_name=list(name), nearby=nearby, distance= list(distance)), by="id_for_filter"], by="id_for_filter")
  
  return(finallist)
}
```

calculate features
------------------

### download context location data

This code can be used to scrape context location data from OpenStreetMaps. First all the features were scraped and then saved in different `.csv` files.

``` r
data(wrld_simpl)

#https://cran.r-project.org/web/packages/osmdata/vignettes/osmdata.html 
#https://ropensci.github.io/osmdata/index.html

# these are the extents of the Netherlands
# we use this to create a box from south to north and east to west to see the coordinates in which we have to find the feaures
matrixNL <- raster::getData('GADM', country="NL", level=0)@bbox

# now we query osm for highways
feature_of_interest <-  opq(bbox = matrixNL, timeout=1000) %>%
  add_osm_feature(key = "highway", value = "motorway") %>%
  osmdata_sf(quiet = FALSE)
# then we filter only thos points within the box that actually fall within in the polygon of the Netherlands
# so we filter the part of Belgium between Limburg and Zeeland for example

#https://datahub.io/core/geo-countries#resource-geo-countries_zip
json_file <- 'countries.geojson'
json_data2 <- fromJSON(json_file)

NL <- as_tibble(json_data2$features)[169,]$geometry$coordinates[[1]]

NL_polygons <- list()

for(i in c(4,9)){
  NL_polygons[[i]] <- as.data.frame(matrix(NL[[i]],  ncol=2))
}


polygonNL <- raster::bind(NL_polygons[[9]],
                          NL_polygons[[4]]) %>%
  as_tibble() %>%
  rowid_to_column("id") %>%
  filter(id < 435 | id >591)%>%
  mutate(V2 = ifelse(id == 434, 53.5, V2))%>%
  mutate(V1 = ifelse(id == 434, 5, V1))%>%
  dplyr::select(-id)%>%
  as.matrix()


#ggplot(polygonNL)+
#  geom_polygon(aes(V1, V2), alpha = .2, fill = "red")

# this function was borrowed from the OSMdata package
trim_to_poly_pts <- function (dat, bb_poly, exclude = TRUE)
{
  if (is (dat$osm_points, 'sf'))
  {
    g <- do.call (rbind, dat$osm_points$geometry)
    indx <- sp::point.in.polygon (g [, 1], g [, 2],
                                  bb_poly [, 1], bb_poly [, 2])
    if (exclude)
      indx <- which (indx == 1)
    else
      indx <- which (indx > 0)
    dat$osm_points <- dat$osm_points [indx, ]
  }
  
  return (dat)
}

feature_of_interest_trimmed <- trim_to_poly_pts(feature_of_interest, polygonNL)

# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed<- as_tibble(feature_of_interest_trimmed$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed$lon <- NA
location_feature_of_interest_trimmed$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed)){
  location_feature_of_interest_trimmed$lon[i] <- location_feature_of_interest_trimmed[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed$lat[i] <- location_feature_of_interest_trimmed[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed <- location_feature_of_interest_trimmed %>%
  dplyr::select(osm_id, lon, lat)

fwrite(location_feature_of_interest_trimmed, "external_imput_data/highwaysNL.csv")


rm(feature_of_interest_trimmed)
rm(feature_of_interest)





# now we query osm for railways
feature_of_interest2 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "railway", value = "rail") %>%
  osmdata_sf(quiet = FALSE)

# then we filter only thos points within the box that actually fall within in the polygon of the Netherlands
# so we filter the part of Belgium between Limburg and Zeeland for example


feature_of_interest_trimmed2 <- trim_to_poly_pts(feature_of_interest2, polygonNL)

# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed2<- as_tibble(feature_of_interest_trimmed2$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed2$lon <- NA
location_feature_of_interest_trimmed2$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed2)){
  location_feature_of_interest_trimmed2$lon[i] <- location_feature_of_interest_trimmed2[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed2$lat[i] <- location_feature_of_interest_trimmed2[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed2 <- location_feature_of_interest_trimmed2 %>%
 dplyr::select(osm_id, lon, lat)

fwrite(location_feature_of_interest_trimmed2, "external_imput_data/traintracksNL.csv")

rm(feature_of_interest_trimmed2)
rm(feature_of_interest2)




feature_of_interest3 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "highway", value = "bus_stop") %>%
  osmdata_sf(quiet = FALSE)

# then we filter only those points within the box that actually fall within in the polygon of the Netherlands
# so we filter the part of Belgium between Limburg and Zeeland for example


feature_of_interest_trimmed3 <- trim_to_poly_pts(feature_of_interest3, polygonNL)

# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed3<- as_tibble(feature_of_interest_trimmed3$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed3$lon <- NA
location_feature_of_interest_trimmed3$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed3)){
  location_feature_of_interest_trimmed3$lon[i] <- location_feature_of_interest_trimmed3[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed3$lat[i] <- location_feature_of_interest_trimmed3[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed3 <- location_feature_of_interest_trimmed3 %>%
  dplyr::select(osm_id, lon, lat, name)

fwrite(location_feature_of_interest_trimmed3, "external_imput_data/busstopsNL.csv")



feature_of_interest4 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "railway", value = "tram") %>%
  osmdata_sf(quiet = FALSE)


feature_of_interest_trimmed4 <- trim_to_poly_pts(feature_of_interest4, polygonNL)


# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed4<- as_tibble(feature_of_interest_trimmed4$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed4$lon <- NA
location_feature_of_interest_trimmed4$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed4)){
  location_feature_of_interest_trimmed4$lon[i] <- location_feature_of_interest_trimmed4[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed4$lat[i] <- location_feature_of_interest_trimmed4[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed4 <- location_feature_of_interest_trimmed4 %>%
  dplyr::select(osm_id, lon, lat)


fwrite(location_feature_of_interest_trimmed4, "external_imput_data/tramtracksNL.csv")


feature_of_interest5 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "railway", value = "subway") %>%
  osmdata_sf(quiet = FALSE)

feature_of_interest_trimmed5 <- trim_to_poly_pts(feature_of_interest5, polygonNL)


# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed5<- as_tibble(feature_of_interest_trimmed5$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed5$lon <- NA
location_feature_of_interest_trimmed5$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed5)){
  location_feature_of_interest_trimmed5$lon[i] <- location_feature_of_interest_trimmed5[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed5$lat[i] <- location_feature_of_interest_trimmed5[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed5 <- location_feature_of_interest_trimmed5 %>%
  dplyr::select(osm_id, lon, lat)

fwrite(location_feature_of_interest_trimmed5, "external_imput_data/subwaytracksNL.csv")

#########



feature_of_interest6 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "station", value = "subway") %>%
  osmdata_sf(quiet = FALSE)

feature_of_interest_trimmed6 <- trim_to_poly_pts(feature_of_interest6, polygonNL)


# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed6<- as_tibble(feature_of_interest_trimmed6$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed6$lon <- NA
location_feature_of_interest_trimmed6$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed6)){
  location_feature_of_interest_trimmed6$lon[i] <- location_feature_of_interest_trimmed6[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed6$lat[i] <- location_feature_of_interest_trimmed6[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed6 <- location_feature_of_interest_trimmed6 %>%
  dplyr::select(osm_id, lon, lat, name)
fwrite(location_feature_of_interest_trimmed6, "external_imput_data/subwaystationNL.csv")

#########


feature_of_interest7 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "railway", value = "tram_stop") %>%
  osmdata_sf(quiet = FALSE)

feature_of_interest_trimmed7 <- trim_to_poly_pts(feature_of_interest7, polygonNL)


# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed7<- as_tibble(feature_of_interest_trimmed7$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed7$lon <- NA
location_feature_of_interest_trimmed7$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed7)){
  location_feature_of_interest_trimmed7$lon[i] <- location_feature_of_interest_trimmed7[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed7$lat[i] <- location_feature_of_interest_trimmed7[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed7 <- location_feature_of_interest_trimmed7 %>%
  dplyr::select(osm_id, lon, lat, name)
fwrite(location_feature_of_interest_trimmed7, "external_imput_data/tramstopsNL.csv")


#########


feature_of_interest8 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "highway", value = "motorway_junction") %>%
  osmdata_sf(quiet = FALSE)

feature_of_interest_trimmed8 <- trim_to_poly_pts(feature_of_interest8, polygonNL)


# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed8<- as_tibble(feature_of_interest_trimmed8$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed8$lon <- NA
location_feature_of_interest_trimmed8$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed8)){
  location_feature_of_interest_trimmed8$lon[i] <- location_feature_of_interest_trimmed8[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed8$lat[i] <- location_feature_of_interest_trimmed8[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed8 <- location_feature_of_interest_trimmed8 %>%
  dplyr::select(osm_id, lon, lat, name)
fwrite(location_feature_of_interest_trimmed8, "external_imput_data/onnofframpsNL.csv")

#########


feature_of_interest9 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "highway", value = "trunk") %>%
  osmdata_sf(quiet = FALSE)

feature_of_interest_trimmed9 <- trim_to_poly_pts(feature_of_interest9, polygonNL)


# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed9<- as_tibble(feature_of_interest_trimmed9$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed9$lon <- NA
location_feature_of_interest_trimmed9$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed9)){
  location_feature_of_interest_trimmed9$lon[i] <- location_feature_of_interest_trimmed9[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed9$lat[i] <- location_feature_of_interest_trimmed9[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed9 <- location_feature_of_interest_trimmed9 %>%
  dplyr::select(osm_id, lon, lat)
fwrite(location_feature_of_interest_trimmed9, "external_imput_data/2ndhighwayNL.csv")

#########


feature_of_interest10 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "highway", value = "trunk_link") %>%
  osmdata_sf(quiet = FALSE)

feature_of_interest_trimmed10 <- trim_to_poly_pts(feature_of_interest10, polygonNL)


# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed10<- as_tibble(feature_of_interest_trimmed10$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed10$lon <- NA
location_feature_of_interest_trimmed10$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed10)){
  location_feature_of_interest_trimmed10$lon[i] <- location_feature_of_interest_trimmed10[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed10$lat[i] <- location_feature_of_interest_trimmed10[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed10 <- location_feature_of_interest_trimmed10 %>%
  dplyr::select(osm_id, lon, lat, name)
fwrite(location_feature_of_interest_trimmed10, "external_imput_data/2ndhighwayonofframpNL.csv")


#########


feature_of_interest11 <-  opq(bbox = matrixNL, timeout=1000, memsize=4e+9 ) %>%
  add_osm_feature(key = "railway", value = "station") %>%
  osmdata_sf(quiet = FALSE)

feature_of_interest_trimmed11 <- trim_to_poly_pts(feature_of_interest11, polygonNL)


# now we extract only the GPS  coordinates of points
location_feature_of_interest_trimmed11<- as_tibble(feature_of_interest_trimmed11$osm_points) %>%
  dplyr::select(osm_id, name, geometry)  


location_feature_of_interest_trimmed11$lon <- NA
location_feature_of_interest_trimmed11$lat <- NA
for(i in 1:nrow(location_feature_of_interest_trimmed11)){
  location_feature_of_interest_trimmed11$lon[i] <- location_feature_of_interest_trimmed11[i,]$geometry[[1]][1]
  location_feature_of_interest_trimmed11$lat[i] <- location_feature_of_interest_trimmed11[i,]$geometry[[1]][2]
}

location_feature_of_interest_trimmed11 <- location_feature_of_interest_trimmed11 %>%
  dplyr::select(osm_id, lon, lat, name)
fwrite(location_feature_of_interest_trimmed11, "external_imput_data/train_stations_NL.csv")
########
```

### plot OSM points

all the scraped OSM point can be plotted like this:

``` r
data("NLD_muni")

NL <- st_union(NLD_muni)
NL2 <- sf::st_transform(NL, "+proj=longlat +datum=WGS84 +no_defs +ellps=WGS84 +towgs84=0,0,0")
#ggplot()+geom_sf(data = NL2, fill=NA, col = "black")


#files <- list.files("./external_imput_data", rec)
listfiles <- list()
for(i in files){
  listfiles[[i]] <- fread(paste0("./external info/", i))
}

df <- rbindlist(listfiles, fill =TRUE, idcol="id")
unique(df$id)
df2 <- df %>%
  mutate(type = ifelse(id == "2ndhighwayNL.csv" , "Highway (n = 23199)",
                       ifelse(id == "highwaysNL.csv" , "Freeway (n = 41201)",
                              ifelse(id == "onnofframpsNL.csv" |id == "2ndhighwayonofframpNL.csv" , "On-off ramp (n = 17617)",
                                     ifelse(id == "busstopsNL.csv", "Bus stop (n = 41872)",
                                            ifelse(id == "subwaystationNL.csv", "Metro stop (n = 426)",
                                                   ifelse(id == "subwaytracksNL.csv", "Metro track (n = 7709)",
                                                          ifelse(id == "tramstopsNL.csv", "Tram stop (n = 1354)",
                                                                 ifelse(id == "tramtracksNL.csv", "Tram track (n = 29674)",
                                                                        ifelse(id == "train_stations_NL.csv", "Train station (n = 769)",
                                                                               "Train track (n = 202792)"))))))))))

df2 %>%
  group_by(type) %>%
  summarise(n = n())

plot_points <- ggplot()+
  geom_sf(data = NL2, fill=NA, col = "black")+
  geom_point(data=df2, aes(x     = lon, 
                           y     = lat, 
                           col   = type,
                           size  = type, 
                           shape = type ))+
  scale_shape_manual(values=c("Highway (n = 23199)"      = 16,
                              "Freeway (n = 41201)"      = 16,
                              "On-off ramp (n = 17617)"  = 16,
                              "Bus stop (n = 41872)"     = 3,
                              "Metro stop (n = 426)"     = 23,
                              "Metro track (n = 7709)"   = 16,
                              "Tram stop (n = 1354)"     = 23,
                              "Tram track (n = 29674)"   = 16,
                              "Train track (n = 202792)" = 16,
                              "Train station (n = 769)"  = 16))+
  scale_color_manual(values=c("Highway (n = 23199)"      = "darkred",
                              "Freeway (n = 41201)"      = "red",
                              "On-off ramp (n = 17617)"  = "yellow",
                              "Bus stop (n = 41872)"     = "grey40",
                              "Metro stop (n = 426)"     = "green",
                              "Metro track (n = 7709)"   = "green",
                              "Tram stop (n = 1354)"     = "purple",
                              "Tram track (n = 29674)"   = "purple",
                              "Train track (n = 202792)" ="blue",
                              "Train station (n = 769)"  = "orange"))+
  scale_fill_manual(values=c("Highway (n = 23199)"       = "darkred",
                             "Freeway (n = 41201)"       = "red",
                             "On-off ramp( n = 17617)"   = "yellow",
                             "Bus stop (n = 41872)"      = "grey40",
                             "Metro stop (n = 426)"      = "green",
                             "Metro track (n = 7709)"    = "green",
                             "Tram stop (n = 1354)"      = "purple",
                             "Tram track (n = 29674)"    = "purple",
                             "Train track (n = 202792)"  ="blue",
                             "Train station (n = 769)"   = "orange"))+
  scale_size_manual(values=c("Highway (n = 23199)"       = 1,
                             "Freeway (n = 41201)"       = 1,
                             "On-off ramp (n = 17617)"   = .8,
                             "Bus stop (n = 41872)"      = .2,
                             "Metro stop (n = 426)"      = .8,
                             "Metro track (n = 7709)"    = .4,
                             "Tram stop (n = 1354)"      = .8,
                             "Tram track (n = 29674)"    = .4,
                             "Train track (n = 202792)"  = .6,
                             "Train station (n = 769)"   = 1))+
  scale_alpha_manual(values=c("Highway (n = 23199)"      = .8,
                              "Freeway (n = 41201)"      =.8,
                              "On-off ramp (n = 17617)"  = .8,
                              "Bus stop (n = 41872)"     = .2,
                              "Metro stop (n = 426)"     = .6,
                              "Metro track (n = 7709)"   = .6,
                              "Tram stop (n = 1354)"     = .2,
                              "Tram track (n = 29674)"   = .2,
                              "Train track (n = 202792)" =.8,
                              "Train station (n = 769)"  = .8))+
  theme_map()+
  theme(legend.position = c(.9, 0.1),
        panel.grid.major = element_line(colour = 'transparent'),
        #panel.spacing.x = unit(4, "mm"),
        legend.background = element_blank(),
        legend.box.background = element_blank(),
        legend.key = element_blank())+
  labs(col = "",
       alpha = "",
       fill = "",
       size = "",
       shape = "")



ggsave("plot_points2.png", plot_points)
```

### load in context location data

Since, all the context features, were saved as separate files, we can now use this code to read them in again.

``` r
station.data  <- fread("./external_imput_data/train_stations_NL.csv")
bus.stop.data <- fread("./external_imput_data/busstopsNL.csv")
highways.data <- bind_rows(fread("./external_imput_data/2ndhighwayNL.csv"), fread("./external_imput_data/highwaysNL.csv"))
onnofframpsNL.data   <- bind_rows(fread("./external_imput_data/onnofframpsNL.csv"), fread("./external_imput_data/2ndhighwayonofframpNL.csv"))
subwaystationNL.data <- fread("./external_imput_data/subwaystationNL.csv") 
subwaytracksNL.data  <- fread("./external_imput_data/subwaytracksNL.csv") 
tramtracksNL.data    <- fread("./external_imput_data/tramtracksNL.csv") 
tramstopsNL.data     <- fread("./external_imput_data/tramstopsNL.csv") 
traintracksNL.data   <- fread("./external_imput_data/traintracksNL.csv") 
```

### load in the CBS municipality and urbanizity data

Statistics Netherlands collects data on population density at the municipality level and has shape files of all the urban centres of the Netherlands. These files are available and can be downloaded at these addresses:

-   <https://www.cbs.nl/nl-nl/dossier/nederland-regionaal/geografische%20data/wijk-en-buurtkaart-2017>
-   <http://download.cbs.nl/regionale-kaarten/2011-bevolkingskern-shape.zip>

``` r
buurt  <- readOGR(dsn="external_imput_data/buurt_dens/buurt_2017.shp")
buurt2 <- spTransform(buurt, CRS("+proj=longlat"))
buurt3 <- fortify(buurt2, region = "BU_CODE")

buurt4 <- left_join(buurt3, 
                    data.frame("id"= as.factor(buurt$BU_CODE),  
                               "STED"= as.character(buurt$STED)
                               ,"BEV_DICHTH"= as.numeric(buurt$BEV_DICHTH)
                    ), 
                    by="id") %>%
  mutate(STED = ifelse(STED == "-99999999", NA_character_, STED)) %>%
  mutate(BEV_DICHTH = ifelse(BEV_DICHTH < 1, NA, BEV_DICHTH))


bevolkingskern  <- readOGR(dsn="external_imput_data/bevolkings_kern/bevolkingskern_2011.shp")
bevolkingskern2 <- spTransform(bevolkingskern, CRS("+proj=longlat"))
bevolkingskern3 <- fortify(bevolkingskern2, region = "KERN_CODE")

bevolkingskern4 <- left_join(bevolkingskern3, 
                    data.frame("id"= as.factor(bevolkingskern$KERN_CODE),  
                               "STED"= as.character(bevolkingskern$STED)
                    ), 
                    by="id") %>%
  mutate(STED = ifelse(STED == "-99999999", NA_character_, STED))
```

### loop over all the trips and create relevant feature statistics per track

This codes loads in, 1 by 1, all the trips and then for all of them calculates the statistics on the features. This process has to be repeated four times. once for the labelled tracks without interpolation and once with. The same goes for the unlabelled tracks.

``` r
files<-list.files("output_files/RDS_per_track_merged_tracks")
already_done <- gsub('.{4}$', '',substr(list.files("./output_files/filtered_merged_tracks_stats"),10,1000))


for(i in files[!(files%in%already_done)]){
  tryCatch({
    file <- readRDS(paste0("./output_files/RDS_per_track_merged_tracks/",i))
    
    print(paste0("starting:", i))
    
        setDT(file)
    
    file_unique <- unique(file, by= c("device_id",
                                      "TrackDataIdVar", 
                                      "start_time",  
                                      "end_time",
                                      "local_track_id",
                                      "mode3", 
                                      "latitude", 
                                      "longitude", 
                                      "accuracy", 
                                      "altitude",  
                                      "timestamp_location"
                                      #,
                                      #"TrackDataIdVar.copy"
    ))
    
    
    
    output.smoothed.cleaned <- GPSdatacleaner(GPSdataframe = file_unique,
                                              do_filter_speed_accuracy=TRUE,
                                              station.data   = NULL,
                                              des.acc        = 200,
                                              des.max.speed  = 250,
                                              long_time_diff = 1000,
                                              impute         = TRUE, 
                                              impute.NS      = FALSE,
                                              distance.hitbox.trainstation = .2,
                                              Savitzky.Golay = TRUE,
                                              Savitzky.Golay.p = 3,
                                              Savitzky.Golay.n = 101, 
                                              Kalman.filter.accuracy = TRUE,
                                              interpolate = FALSE,
                                              R_mult.Kalman_filter_accuracy = 2,
                                              Q_mult.Kalman_filter_accuracy = 1.5,
                                              Kalman.filter.smooth = FALSE,
                                              R_value_smooth_Kalman = 10)
    
    
    distance_straight_line <- distVincentySphere(matrix(c(output.smoothed.cleaned$longitude_imputedT[1],  
                                                          output.smoothed.cleaned$latitude_imputedT[1]), 
                                                        ncol=2),
                                                 matrix(c(output.smoothed.cleaned$longitude_imputedT[nrow(output.smoothed.cleaned)],
                                                          output.smoothed.cleaned$latitude_imputedT[nrow(output.smoothed.cleaned)]),
                                                        ncol=2))
    
    
    
    
    total_distance <- output.smoothed.cleaned %>%
      arrange(time_forced_copy) %>%
      mutate(lag.latitude_imputedT  = dplyr::lag(latitude_imputedT, n = 1, default = NA))  %>%
      mutate(lag.longitude_imputedT = dplyr::lag(longitude_imputedT, n = 1, default = NA)) %>%
      mutate(lag.longitude_imputedT = if_else(is.na(lag.longitude_imputedT), longitude_imputedT, lag.longitude_imputedT)) %>%
      mutate(lag.latitude_imputedT = if_else(is.na(lag.latitude_imputedT), latitude_imputedT, lag.latitude_imputedT))    %>%
      mutate(distance.between.points2 = distVincentySphere(matrix(c(longitude_imputedT, latitude_imputedT), ncol=2), 
                                                           matrix(c(lag.longitude_imputedT, lag.latitude_imputedT), ncol=2))) %>%
      dplyr::select(distance.between.points2 )%>%
      summarise(total_distance = sum(distance.between.points2))
    
    ratio_straightline_distance <- distance_straight_line/total_distance
    
    total_distance.segments <- output.smoothed.cleaned %>%
      arrange(time_forced_copy) %>%
      group_by(segment_id)%>%
      mutate(lag.latitude_imputedT  = dplyr::lag(latitude_imputedT, n = 1, default = NA))  %>%
      mutate(lag.longitude_imputedT = dplyr::lag(longitude_imputedT, n = 1, default = NA)) %>%
      mutate(lag.longitude_imputedT = if_else(is.na(lag.longitude_imputedT), longitude_imputedT, lag.longitude_imputedT)) %>%
      mutate(lag.latitude_imputedT = if_else(is.na(lag.latitude_imputedT), latitude_imputedT, lag.latitude_imputedT))    %>%
      mutate(distance.between.points2 = distVincentySphere(matrix(c(longitude_imputedT, latitude_imputedT), ncol=2), 
                                                           matrix(c(lag.longitude_imputedT, lag.latitude_imputedT), ncol=2))) %>%
      dplyr::select(segment_id, distance.between.points2 )%>%
      summarise(total_distance = sum(distance.between.points2)) %>%
      ungroup()%>%
      summarise(total_distance.segments = sum(total_distance))
    
    
    
    average_Speed_track <- output.smoothed.cleaned %>%
      summarise(diff_time_total = as.numeric(time_forced_copy[nrow(output.smoothed.cleaned)] - time_forced_copy[1],"secs"),
                average_Speed_track = total_distance$total_distance/diff_time_total)
    
    
    
    speed_stats <- output.smoothed.cleaned %>%
      arrange(time_forced_copy) %>%
      mutate(lag.latitude_imputedT  = dplyr::lag(latitude_imputedT, n = 1, default = NA))  %>%
      mutate(lag.longitude_imputedT = dplyr::lag(longitude_imputedT, n = 1, default = NA)) %>%
      mutate(lag.longitude_imputedT = if_else(is.na(lag.longitude_imputedT), longitude_imputedT, lag.longitude_imputedT)) %>%
      mutate(lag.latitude_imputedT = if_else(is.na(lag.latitude_imputedT), latitude_imputedT, lag.latitude_imputedT))    %>%
      mutate(distance.between.points2 = distVincentySphere(matrix(c(longitude_imputedT, latitude_imputedT), ncol=2), 
                                                           matrix(c(lag.longitude_imputedT, lag.latitude_imputedT), ncol=2))) %>%
      mutate(lag.time_forced_copy  = dplyr::lag(time_forced_copy, n = 1, default = NA))  %>%
      mutate(lag.time_forced_copy = if_else(is.na(lag.time_forced_copy), time_forced_copy, lag.time_forced_copy)) %>%
      mutate(diff_time = as.numeric(time_forced_copy - lag.time_forced_copy,"secs")) %>%
      mutate(speed_between_points = distance.between.points2/diff_time)%>%
      mutate(acceleration   = c(0, diff(speed_between_points,1)) / diff_time) %>%
      mutate(bearing        = geosphere::bearing(matrix(c(longitude_imputedT, latitude_imputedT), ncol=2), 
                                                 matrix(c(lag.longitude_imputedT, lag.latitude_imputedT), ncol=2))) %>%
      mutate(abs.bearing    = if_else(bearing == -180, 0, abs(bearing)))          %>%
      mutate(rel_bearing    = abs(abs.bearing - dplyr::lag(abs.bearing, n = 1, default = NA))) %>% 
      dplyr::select(distance.between.points2, lag.time_forced_copy,  lag.time_forced_copy, diff_time, speed_between_points, acceleration, bearing, abs.bearing, rel_bearing)%>%
      summarise( mean_bearing = mean(bearing, na.rm=TRUE),
                 median_bearing = median(bearing, na.rm=TRUE),
                 max_bearing = max(bearing, na.rm=TRUE),
                 bearing_0.05 = quantile(bearing, 0.05, na.rm=TRUE), 
                 bearing_0.25 = quantile(bearing, 0.25, na.rm=TRUE), 
                 bearing_0.75 = quantile(bearing, 0.75, na.rm=TRUE), 
                 bearing_0.9 = quantile(bearing, 0.9, na.rm=TRUE), 
                 bearing_0.95 = quantile(bearing, 0.95, na.rm=TRUE),
                 bearing_0.99 = quantile(bearing, 0.99, na.rm=TRUE),
                 bearing_sd = sd(bearing, na.rm=TRUE),
                 mean_abs.bearing = mean(abs.bearing, na.rm=TRUE),
                 median_abs.bearing = median(abs.bearing, na.rm=TRUE),
                 max_abs.bearing = max(abs.bearing, na.rm=TRUE),
                 abs.bearing_0.05 = quantile(abs.bearing, 0.05, na.rm=TRUE), 
                 abs.bearing_0.25 = quantile(abs.bearing, 0.25, na.rm=TRUE), 
                 abs.bearing_0.75 = quantile(abs.bearing, 0.75, na.rm=TRUE), 
                 abs.bearing_0.9 = quantile(abs.bearing, 0.9, na.rm=TRUE), 
                 abs.bearing_0.95 = quantile(abs.bearing, 0.95, na.rm=TRUE),
                 abs.bearing_0.99 = quantile(abs.bearing, 0.99, na.rm=TRUE),
                 abs.bearing_sd = sd(abs.bearing, na.rm=TRUE),
                 mean_rel_bearing = mean(rel_bearing, na.rm=TRUE),
                 median_rel_bearing = median(rel_bearing, na.rm=TRUE),
                 max_rel_bearing = max(rel_bearing, na.rm=TRUE),
                 rel_bearing_0.05 = quantile(rel_bearing, 0.05, na.rm=TRUE), 
                 rel_bearing_0.25 = quantile(rel_bearing, 0.25, na.rm=TRUE), 
                 rel_bearing_0.75 = quantile(rel_bearing, 0.75, na.rm=TRUE), 
                 rel_bearing0.9 = quantile(rel_bearing, 0.9, na.rm=TRUE), 
                 rel_bearing_0.95 = quantile(rel_bearing, 0.95, na.rm=TRUE),
                 rel_bearing_0.99 = quantile(rel_bearing, 0.99, na.rm=TRUE),
                 rel_bearing_sd = sd( rel_bearing, na.rm=TRUE),
                 mean_speed = mean(speed_between_points[is.finite(speed_between_points)], na.rm=TRUE),
                 median_speed = median(speed_between_points[is.finite(speed_between_points)], na.rm=TRUE),
                 max_speed = max(speed_between_points[is.finite(speed_between_points)], na.rm=TRUE),
                 speed_0.05 = quantile(speed_between_points[is.finite(speed_between_points)], 0.05, na.rm=TRUE), 
                 speed_0.25 = quantile(speed_between_points[is.finite(speed_between_points)], 0.25, na.rm=TRUE), 
                 speed_0.75 = quantile(speed_between_points[is.finite(speed_between_points)], 0.75, na.rm=TRUE), 
                 speed_0.9  = quantile(speed_between_points[is.finite(speed_between_points)], 0.9, na.rm=TRUE), 
                 speed_0.95 = quantile(speed_between_points[is.finite(speed_between_points)], 0.95, na.rm=TRUE),
                 speed_0.99 = quantile(speed_between_points[is.finite(speed_between_points)], 0.99, na.rm=TRUE),
                 speed_sd = sd(speed_between_points[is.finite(speed_between_points)], na.rm=TRUE),
                 speed_skew = skewness(speed_between_points[is.finite(speed_between_points)], na.rm=TRUE),
                 speed_above80_ratio = sum(speed_between_points[is.finite(speed_between_points)]>22.2)/length(speed_between_points[is.finite(speed_between_points)]),
                 speed_above120_ratio = sum(speed_between_points[is.finite(speed_between_points)]>33.3)/length(speed_between_points[is.finite(speed_between_points)]),
                 speed_below5_ratio = sum(speed_between_points[is.finite(speed_between_points)]<1.38)/length(speed_between_points[is.finite(speed_between_points)]), 
                 n_inf_acc = sum(acceleration == Inf, na.rm=TRUE),
                 mean_acceleration = mean(acceleration[is.finite(acceleration)], na.rm=TRUE),
                 median_acceleration = median(acceleration[is.finite(acceleration)], na.rm=TRUE),
                 max_acceleration = max(acceleration[is.finite(acceleration)], na.rm=TRUE),
                 min_acceleration = min(acceleration[is.finite(acceleration)], na.rm=TRUE),
                 acceleration_0.01 = quantile(acceleration[is.finite(acceleration)], 0.01, na.rm=TRUE), 
                 acceleration_0.05 = quantile(acceleration[is.finite(acceleration)], 0.05, na.rm=TRUE),
                 acceleration_0.1 = quantile(acceleration[is.finite(acceleration)], 0.1, na.rm=TRUE), 
                 acceleration_0.25 = quantile(acceleration[is.finite(acceleration)], 0.25, na.rm=TRUE), 
                 acceleration_0.75 = quantile(acceleration[is.finite(acceleration)], 0.75, na.rm=TRUE), 
                 acceleration_0.9 = quantile(acceleration[is.finite(acceleration)], 0.9, na.rm=TRUE), 
                 acceleration_0.95 = quantile(acceleration[is.finite(acceleration)], 0.95, na.rm=TRUE),
                 acceleration_0.99 = quantile(acceleration[is.finite(acceleration)], 0.99, na.rm=TRUE),
                 acceleration_sd = sd(acceleration[is.finite(acceleration)], na.rm=TRUE),
                 acceleration_skew = skewness(acceleration[is.finite(acceleration)], na.rm=TRUE))
    
    # apart from bearing we can also calculate how man sharp turns +90  and +120, there are in the track per data point and per second

    if(nrow(output.smoothed.cleaned)<3){
      angle_stats<- tibble(mean_turn=0,
                           median_turn= 0,
                           n_turn_1plus90 = 0,
                           n_turn_1plus150 = 0,
                           n_turn_5plus90 = 0,
                           n_turn_5plus150 =0,
                           turn_1plus90_ps = 0,
                           turn_1plus150_ps =0,
                           turn_5plus90_ps = 0,
                           turn_5plus150_ps = 0)
    }else{

      angle <- function(pt1, pt2) {
        conv = 180 / pi # conversion radians / degrees
        diff <- pt2 - pt1
        left <- sign(diff[1]) # is X diff < 0?
        left[left == 0] <- -1 # sign(0) = 0, need to keep as nonzero
        angle <- left * (diff[2] * pi + atan(diff[1] / diff[2]))
        angle[angle < 0] <- angle[angle < 0] + 2 * pi
        return(angle * conv)
      }

      gpx.trackpoints<-data.frame(lon = output.smoothed.cleaned$longitude_imputedT,
                                  lat = output.smoothed.cleaned$latitude_imputedT)

      gpx.trackpoints <- na.omit(gpx.trackpoints)
      gpx.tp <- coordinates(gpx.trackpoints)

      for (tp in 1:(nrow(gpx.tp)-1)) {
        gpx.trackpoints$dist[tp] <- raster::pointDistance(gpx.tp[tp,], gpx.tp[tp+1,], lonlat=T)
      }
      for (tp in 1:(nrow(gpx.tp)-1)) {
        gpx.trackpoints$course[tp] <- angle(gpx.tp[tp + 1,], gpx.tp[tp,])
        gpx.trackpoints$course <- ifelse(is.na(gpx.trackpoints$course), zoo::na.approx(gpx.trackpoints$course), gpx.trackpoints$course)
      }
      for (tp in 1:nrow(gpx.tp)) {
        gpx.trackpoints$turn1[tp] <- abs(gpx.trackpoints$course[tp] - gpx.trackpoints$course[tp +1])
        gpx.trackpoints$turn1[tp] <- ifelse(is.na(gpx.trackpoints$turn1[tp]), 0, gpx.trackpoints$turn1[tp])
        gpx.trackpoints$turn1[tp] <- ifelse(gpx.trackpoints$turn1[tp] > 180, 360 - gpx.trackpoints$turn1[tp], gpx.trackpoints$turn1[tp])

        gpx.trackpoints$turn5[tp] <- abs(gpx.trackpoints$course[tp] - gpx.trackpoints$course[tp + 5]) # if we just look one data point ahead, we dont see that many angles if we actually have measure only one gps coordinate ahead, so we take 5 ahead as well.
        gpx.trackpoints$turn5[tp] <- ifelse(is.na(gpx.trackpoints$turn5[tp]), 0, gpx.trackpoints$turn5[tp])
        gpx.trackpoints$turn5[tp] <- ifelse(gpx.trackpoints$turn5[tp] > 180, 360 - gpx.trackpoints$turn5[tp], gpx.trackpoints$turn5[tp])
      }

      angle_stats<- gpx.trackpoints %>%
        as_tibble() %>%
        summarize(mean_dist_betweenpoint = mean(dist, na.rm=T),
                  mean_turn= mean(turn1, na.rm=T),
                  median_turn= median(turn1, na.rm=T),
                  n_turn_1plus90 = sum(turn1>90, na.rm=T),
                  n_turn_1plus150 = sum(turn1>150, na.rm=T),
                  n_turn_5plus90 = sum(turn5>90, na.rm=T),
                  n_turn_5plus150 = sum(turn5>150, na.rm=T),
                  turn_1plus90_ps = n_turn_1plus90/average_Speed_track$diff_time_total,
                  turn_1plus150_ps = n_turn_1plus150/average_Speed_track$diff_time_total,
                  turn_5plus90_ps = n_turn_5plus90/average_Speed_track$diff_time_total,
                  turn_5plus150_ps = n_turn_5plus150/average_Speed_track$diff_time_total)

    }
  
    
    output.smoothed.cleaned <- output.smoothed.cleaned %>%
      mutate(id_for_filter = 1:nrow(output.smoothed.cleaned))
    
    
    dataset1.gps <- output.smoothed.cleaned %>%
      as_tibble()  %>%
      rename("latitude.orig"= latitude,
             "longitude.orig"= longitude) %>%
      rename("latitude"= latitude_imputedT,
             "longitude" =longitude_imputedT)
    
    
    dataset2.subwaytracks <- subwaytracksNL.data %>%
      as_tibble()  %>%
      mutate(id= 1:nrow(subwaytracksNL.data))%>%
      mutate(osm_id = as.character(osm_id))%>%
      rename("name"= osm_id,
             "latitude"= lat,
             "longitude" =lon)  
    
    
    dataset2.tramtracks <- tramtracksNL.data %>%
      as_tibble()  %>%
      mutate(id= 1:nrow(tramtracksNL.data))%>%
      mutate(osm_id = as.character(osm_id))%>%
      rename("name"= osm_id,
             "latitude"= lat,
             "longitude" =lon)  
    
    dataset2.tramstops <- tramstopsNL.data%>%
      as_tibble()  %>%
      mutate(id= 1:nrow(tramstopsNL.data))%>%
      mutate(osm_id = as.character(osm_id))%>%
      dplyr::select(-name)  %>%
      rename("name"= osm_id,
             "latitude"= lat,
             "longitude" =lon)
    
    dataset2.subwaystation <- subwaystationNL.data%>%
      as_tibble()  %>%
      mutate(id= 1:nrow(subwaystationNL.data))%>%
      mutate(osm_id = as.character(osm_id))%>%
      dplyr::select(-name)  %>%
      rename("name"= osm_id,
             "latitude"= lat,
             "longitude" =lon)
    
    
    
    
    #### on-off ramps
    dataset2.onnofframps<- onnofframpsNL.data%>%
      as_tibble()  %>%
      mutate(id= 1:nrow(onnofframpsNL.data))%>%
      mutate(osm_id = as.character(osm_id))%>%
      rename("name"= osm_id,
             "latitude"= lat,
             "longitude" =lon)  
    
    
    ### highways
    dataset2.highways <- highways.data %>%
      as_tibble()  %>%
      mutate(id= 1:nrow(highways.data))%>%
      mutate(osm_id = as.character(osm_id))%>%
      rename("name"= osm_id,
             "latitude"= lat,
             "longitude" =lon)  
    
    ### train tracks
    dataset2.traintracks <- traintracksNL.data %>%
      as_tibble()  %>%
      mutate(id= 1:nrow(traintracksNL.data))%>%
      mutate(osm_id = as.character(osm_id))%>%
      rename("name"= osm_id,
             "latitude"= lat,
             "longitude" =lon)  
    ### train station
    dataset2.station <- station.data %>%
      as_tibble()  %>%
      mutate(id= 1:nrow(station.data))%>%
      rename("name"= name,
             "latitude"= lat,
             "longitude" =lon)%>%
      dplyr::select(-osm_id)
    
    ### bus stops
    dataset2.bus_stop <- bus.stop.data %>%
      as_tibble()  %>%
      mutate(id= 1:nrow(bus.stop.data))%>%
      rename("name"= name,
             "latitude"= lat,
             "longitude" =lon)%>%
      dplyr::select(-osm_id)
  
    

    list_datasets <-   split(dataset1.gps, dataset1.gps$id_for_filter %/% 250)
    
    output.subwaytracks_list <- list()
    output.tramtracks_list <- list()
    output.tram_stop_list <- list()
    output.subway_station_list <- list()
    output.onofframps_list <- list()
    output.highways_list <- list()
    output.traintracks_list <- list()
    output.station_list <- list()
    output.bus_stops_list <- list()
    
    
    
    
    for(g in 1:length(list_datasets)){
      dataset1.gps  <- list_datasets[[g]]
      output.subwaytracks_list[[g]]  <- check_if_close_fast(dataset1 = dataset1.gps,
                                                            dataset2 = dataset2.subwaytracks, 
                                                            desired.dist = 0.05,
                                                            margin       = 100,
                                                            margin2      = 100)
      
      output.tramtracks_list[[g]]  <- check_if_close_fast(dataset1 = dataset1.gps,
                                                          dataset2 = dataset2.tramtracks, 
                                                          desired.dist = 0.05,
                                                          margin       = 100,
                                                          margin2      = 100)
      
      
      output.tram_stop_list[[g]]  <- check_if_close2(dataset1 = dataset1.gps,
                                                     dataset2 = dataset2.tramstops, 
                                                     desired.dist = 0.1,
                                                     margin       = .005,
                                                     margin2      = .005)
      output.subway_station_list[[g]]      <- check_if_close2(dataset1 = dataset1.gps,
                                                              dataset2 = dataset2.subwaystation, 
                                                              desired.dist = 0.1,
                                                              margin       = .005,
                                                              margin2      = .005)  
      output.onofframps_list[[g]]  <- check_if_close2(dataset1 = dataset1.gps,
                                                      dataset2 = dataset2.onnofframps, 
                                                      desired.dist = 0.1,
                                                      margin       = .005,
                                                      margin2      = .005) 
      output.highways_list[[g]]  <- check_if_close_fast(dataset1 = dataset1.gps,
                                                        dataset2 = dataset2.highways, 
                                                        desired.dist = 0.05,
                                                        margin       = 100,
                                                        margin2      = 100)
      
      output.traintracks_list[[g]]  <- check_if_close_fast(dataset1 = dataset1.gps,
                                                           dataset2 = dataset2.traintracks, 
                                                           desired.dist = 0.075,
                                                           margin       = 100,
                                                           margin2      = 100)
      
      output.station_list[[g]] <- check_if_close2(dataset1 = dataset1.gps,
                                                  dataset2 = dataset2.station, 
                                                  desired.dist = 0.2,
                                                  margin       = .005,
                                                  margin2      = .005)
      output.bus_stops_list[[g]] <- check_if_close2(dataset1 = dataset1.gps,
                                                    dataset2 = dataset2.bus_stop, 
                                                    desired.dist = 0.075,
                                                    margin       = .005,
                                                    margin2      = .005)
      
      
      
      
      print(paste0(g, "/", length(list_datasets)))
    }
    output.subwaytracks <-  do.call(rbind, output.subwaytracks_list)
    output.tramtracks   <-  do.call(rbind, output.tramtracks_list)
    output.tram_stop    <-  do.call(rbind, output.tram_stop_list)
    output.subway_station <- do.call(rbind, output.subway_station_list)
    output.onofframps <-   do.call(rbind, output.onofframps_list)
    output.highways <- do.call(rbind, output.highways_list)
    output.traintracks <- do.call(rbind, output.traintracks_list)
    output.station <- do.call(rbind, output.station_list)
    output.bus_stops <- do.call(rbind, output.bus_stops_list)
  
    
    # subway tracks
    
    stats_nearby_subway_tracks <- output.smoothed.cleaned %>%
      as_tibble()%>%
      dplyr::select(latitude_imputedT, 
                    longitude_imputedT, 
                    time_forced_copy, 
                    device_id, 
                    #TrackDataIdVar,
                    id,
                    id_for_filter)%>%    
      left_join(output.subwaytracks, by = "id_for_filter") %>%
      arrange(time_forced_copy) %>%
      summarise(gpsPoints_nearby_subway_track      =  sum(nearby, na.rm= TRUE),
                mean_distance_subway_track    = mean(distance, na.rm= TRUE),
                median_distance_subway_track   = median(distance, na.rm= TRUE),
                near_subwaytrack_relative1 = gpsPoints_nearby_subway_track / average_Speed_track$diff_time_total, # per second
                near_subwaytrack_relative2 = gpsPoints_nearby_subway_track / nrow(output.smoothed.cleaned)) # per data point
    
    
    
    
    #### tram tracks
    
    
    
    stats_nearby_tram_tracks<- output.smoothed.cleaned %>%
      as_tibble()%>%
      dplyr::select(latitude_imputedT, 
                    longitude_imputedT, 
                    time_forced_copy, 
                    device_id, 
                    #TrackDataIdVar,
                    id,
                    id_for_filter)%>%    
      left_join(output.tramtracks, by = "id_for_filter") %>%
      arrange(time_forced_copy) %>%
      summarise(gpsPoints_nearby_tram_track = sum(nearby, na.rm = TRUE),
                mean_distance_tram_track    = mean(distance, na.rm = TRUE),
                median_distance_tram_track  = median(distance, na.rm = TRUE),
                near_tramtracks_relative1  = gpsPoints_nearby_tram_track / average_Speed_track$diff_time_total, # per second
                near_tramtracks_relative2  = gpsPoints_nearby_tram_track / nrow(output.smoothed.cleaned)) # per data point
    
    
    
    #### tram stops
    
    
    
    
    stats_nearby_tram_stop<- output.smoothed.cleaned %>%
      as_tibble()%>%
      dplyr::select(latitude_imputedT, 
                    longitude_imputedT, 
                    time_forced_copy, 
                    device_id, 
                    #TrackDataIdVar,
                    id,
                    id_for_filter)%>%    
      mutate(seconds_since_start = time_forced_copy- min(output.smoothed.cleaned$time_forced_copy))%>%
      mutate(seconds_from_finish = max(output.smoothed.cleaned$time_forced_copy) - time_forced_copy)%>%
      left_join(output.tram_stop, by = "id_for_filter") %>%
      arrange(time_forced_copy) %>%
      summarise(gpsPoints_nearby_tram_stop       =  sum(nearby, na.rm= TRUE),
                mean_distance_tram_stop     = mean(distance, na.rm= TRUE),
                median_distance_tram_stop    = median(distance, na.rm= TRUE),
                gpsPoints_nearby_tram_stop_start = ifelse(sum(nearby[seconds_since_start<300],na.rm= TRUE)!=0, TRUE , FALSE),
                gpsPoints_nearby_tram_stop_end   = ifelse(sum(nearby[seconds_from_finish<300],na.rm= TRUE)!=0, TRUE , FALSE),
                gpsPoints_nearby_tram_stop_start_and_end = ifelse(gpsPoints_nearby_tram_stop_start == TRUE &
                                                                    gpsPoints_nearby_tram_stop_end == TRUE,  TRUE , FALSE),
                unique_tram_stop = length(unique(na.omit(unlist(feature_name)))),
                near_tram_stop_relative1 = gpsPoints_nearby_tram_stop / average_Speed_track$diff_time_total, # per second
                near_tram_stop_relative2 = gpsPoints_nearby_tram_stop / nrow(output.smoothed.cleaned)) # per data point
  
    
    #### subway station 
    

    stats_nearby_subway_station <- output.smoothed.cleaned %>%
      as_tibble()%>%
      dplyr::select(latitude_imputedT, 
                    longitude_imputedT, 
                    time_forced_copy, 
                    device_id, 
                    #TrackDataIdVar,
                    id,
                    id_for_filter)%>%    
      mutate(seconds_since_start = time_forced_copy- min(output.smoothed.cleaned$time_forced_copy))%>%
      mutate(seconds_from_finish = max(output.smoothed.cleaned$time_forced_copy) - time_forced_copy)%>%
      left_join(output.subway_station, by = "id_for_filter") %>%
      arrange(time_forced_copy) %>%
      summarise(gpsPoints_nearby_subway_station       =  sum(nearby, na.rm= TRUE),
                mean_distance_subway_station     = mean(distance, na.rm= TRUE),
                median_distance_subway_station    = median(distance, na.rm= TRUE),
                gpsPoints_nearby_subway_station_start = ifelse(sum(nearby[seconds_since_start<300],na.rm= TRUE)!=0, TRUE , FALSE),
                gpsPoints_nearby_subway_station_end   = ifelse(sum(nearby[seconds_from_finish<300],na.rm= TRUE)!=0, TRUE , FALSE),
                gpsPoints_nearby_subwaystation_start_and_end = ifelse(gpsPoints_nearby_subway_station_start == TRUE &
                                                                        gpsPoints_nearby_subway_station_end == TRUE,  TRUE , FALSE),
                unique_subway_stations = length(unique(na.omit(unlist(feature_name)))),
                near_subway_station_relative1 = gpsPoints_nearby_subway_station / average_Speed_track$diff_time_total, # per second
                near_subway_station_relative2 = gpsPoints_nearby_subway_station / nrow(output.smoothed.cleaned)) # per data point
    
    
    
    
    stats_nearby_onofframps <- output.smoothed.cleaned %>%
      as_tibble()%>%
      dplyr::select(latitude_imputedT, 
                    longitude_imputedT, 
                    time_forced_copy, 
                    device_id, 
                    #TrackDataIdVar,
                    id,
                    id_for_filter)%>%    
      mutate(seconds_since_start = time_forced_copy- min(output.smoothed.cleaned$time_forced_copy))%>%
      mutate(seconds_from_finish = max(output.smoothed.cleaned$time_forced_copy) - time_forced_copy)%>%
      left_join(output.onofframps, by = "id_for_filter") %>%
      arrange(time_forced_copy) %>%
      summarise(gpsPoints_nearby_onofframps       =  sum(nearby, na.rm= TRUE),
                mean_distance_onofframps     = mean(distance, na.rm= TRUE),
                median_distance_onofframps    = median(distance, na.rm= TRUE),
                unique_train_onofframps = length(unique(na.omit(unlist(feature_name)))),
                near_onofframps_relative1 = gpsPoints_nearby_onofframps / average_Speed_track$diff_time_total, # per second
                near_onofframps_relative2 = gpsPoints_nearby_onofframps / nrow(output.smoothed.cleaned)) # per data point
    
    
    

    
    stats_nearby_highways<- output.smoothed.cleaned %>%
      as_tibble()%>%
      dplyr::select(latitude_imputedT, 
                    longitude_imputedT, 
                    time_forced_copy, 
                    device_id, 
                    #TrackDataIdVar,
                    id,
                    id_for_filter)%>%    
      left_join(output.highways, by = "id_for_filter") %>%
      arrange(time_forced_copy) %>%
      summarise(gpsPoints_nearby_highways    =  sum(nearby, na.rm= TRUE),
                mean_distance_highways     = mean(distance, na.rm = TRUE),
                median_distance_highways  = median(distance, na.rm = TRUE),
                near_highways_relative1 = gpsPoints_nearby_highways / average_Speed_track$diff_time_total, # per second
                near_highways_relative2 = gpsPoints_nearby_highways / nrow(output.smoothed.cleaned)) # per data point
    
    
    stats_nearby_train_tracks<- output.smoothed.cleaned %>%
      as_tibble()%>%
      dplyr::select(latitude_imputedT, 
                    longitude_imputedT, 
                    time_forced_copy, 
                    device_id, 
                    #TrackDataIdVar,
                    id,
                    id_for_filter)%>%    
      left_join(output.traintracks, by = "id_for_filter") %>%
      arrange(time_forced_copy) %>%
      summarise(gpsPoints_nearby_train_track      =  sum(nearby, na.rm= TRUE),
                mean_distance_traintracks     = mean(distance, na.rm = TRUE),
                median_distance_traintracks  = median(distance, na.rm = TRUE),
                near_traintracks_relative1 = gpsPoints_nearby_train_track / average_Speed_track$diff_time_total, # per second
                near_traintracks_relative2 = gpsPoints_nearby_train_track / nrow(output.smoothed.cleaned)) # per data point
    
    
    
    stats_nearby_station<- output.smoothed.cleaned %>%
      as_tibble()%>%
      dplyr::select(latitude_imputedT, 
                    longitude_imputedT, 
                    time_forced_copy, 
                    device_id, 
                    #TrackDataIdVar,
                    id,
                    id_for_filter)%>%    
      mutate(seconds_since_start = time_forced_copy- min(output.smoothed.cleaned$time_forced_copy))%>%
      mutate(seconds_from_finish = max(output.smoothed.cleaned$time_forced_copy) - time_forced_copy)%>%
      left_join(output.station, by = "id_for_filter") %>%
      arrange(time_forced_copy) %>%
      summarise(gpsPoints_nearby_train_station       =  sum(nearby, na.rm= TRUE),
                mean_distance_train_station     = mean(distance, na.rm= TRUE),
                median_distance_train_station    = median(distance, na.rm= TRUE),
                gpsPoints_nearby_train_station_start = ifelse(sum(nearby[seconds_since_start<300],na.rm= TRUE)!=0, TRUE , FALSE),
                gpsPoints_nearby_train_station_end   = ifelse(sum(nearby[seconds_from_finish<300],na.rm= TRUE)!=0, TRUE , FALSE),
                gpsPoints_nearby_station_start_and_end = ifelse(gpsPoints_nearby_train_station_start == TRUE &
                                                                  gpsPoints_nearby_train_station_end == TRUE,  TRUE , FALSE),
                unique_train_stations = length(unique(na.omit(unlist(feature_name)))),
                near_trainstation_relative1 = gpsPoints_nearby_train_station / average_Speed_track$diff_time_total, # per second
                near_trainstation_relative2 = gpsPoints_nearby_train_station / nrow(output.smoothed.cleaned)) # per data point
    
    
    stats_nearby_bus_stops <- output.smoothed.cleaned %>%
      as_tibble()%>%
      dplyr::select(latitude_imputedT, 
                    longitude_imputedT, 
                    time_forced_copy, 
                    device_id, 
                    #TrackDataIdVar,
                    id,
                    id_for_filter)%>%
      mutate(seconds_since_start = time_forced_copy- min(output.smoothed.cleaned$time_forced_copy))%>%
      mutate(seconds_from_finish = max(output.smoothed.cleaned$time_forced_copy) - time_forced_copy)%>%
      left_join(output.bus_stops, by = "id_for_filter") %>%
      arrange(time_forced_copy) %>%
      summarise(gpsPoints_nearby_bus_stops   = sum(nearby, na.rm= TRUE),
                mean_distance_bus_stops      = mean(distance, na.rm= TRUE),
                median_distance_bus_stops    = median(distance, na.rm= TRUE),
                gpsPoints_nearby_bus_stops_start = ifelse(sum(nearby[seconds_since_start<300],na.rm= TRUE) !=0, TRUE , FALSE),
                gpsPoints_nearby_bus_stops_end   = ifelse(sum(nearby[seconds_from_finish<300], na.rm= TRUE) !=0, TRUE , FALSE),
                gpsPoints_nearby_bus_stops_start_and_end = ifelse(gpsPoints_nearby_bus_stops_start == TRUE &
                                                                    gpsPoints_nearby_bus_stops_end == TRUE,  TRUE , FALSE),
                unique_bus_stops  = length(unique(na.omit(unlist(feature_name)))),  # this needs to be different then with the train stations, because you can be near two bus stops at the same time, so just counting unique strings is not sufficient.
                near_bus_stops_relative1 = gpsPoints_nearby_bus_stops / average_Speed_track$diff_time_total,
                near_bus_stops_relative2 = gpsPoints_nearby_bus_stops / nrow(output.smoothed.cleaned)) # per data point
    
    
    
    
    dat <- output.smoothed.cleaned %>%
      select(latitude_imputedT, longitude_imputedT) %>%
      rename(Longitude =longitude_imputedT, 
             Latitude = latitude_imputedT) %>%
      mutate(names= 1:nrow(output.smoothed.cleaned))
    
    coordinates(dat) <- ~ Longitude + Latitude
    proj4string(dat) <- proj4string(buurt2)
    
    my_mode <- function(x){  # function to calculate mode
      ux <- unique(x)
      ux[which.max(tabulate(match(x,ux)))]
    }
    
    
    stat_pop_dens <- over(dat, buurt2) %>%
      select(BEV_DICHTH, STED) %>%
      bind_cols(point = dat$names) %>%
      mutate(BEV_DICHTH = ifelse(BEV_DICHTH==-99999999, NA, BEV_DICHTH))%>%
      mutate(STED = ifelse(STED==-99999999, NA, STED))%>%
      summarise(mean_bev_dens = mean(BEV_DICHTH, na.rm=TRUE),
                median_bev_dens = median(BEV_DICHTH, na.rm=TRUE),
                mean_urban =  mean(as.numeric(STED)-1, na.rm=TRUE),
                sdBEV = sd(BEV_DICHTH, na.rm=TRUE),
                median_urban =  median(as.numeric(STED)-1, na.rm=TRUE),
                mode_urban = my_mode(as.numeric(STED)-1),
                ratio_in_urban = sum(STED == "1"| STED == "2")/n())
    
    stat_urbanizity <- over(dat, bevolkingskern2) %>%
      summarise(n_urban_area = sum(is.na(KERN_CODE)),
                unique_urban_area= length(unique(na.omit(as.character(KERN_CODE)))),
                ratio_urban_area = sum(is.na(KERN_CODE))/n())
    
    
    
    morning_start_rush <- as_datetime("1970-01-01 06:30:00")
    morning_end_rush <- as_datetime("1970-01-01 09:00:00")
    evening_start_rush <- as_datetime("1970-01-01 16:30:00")
    evening_end_rush <- as_datetime("1970-01-01 19:00:00")
    
    data_needed <- output.smoothed.cleaned %>%
      summarise(n_segments = n_distinct(segment_id),
                device_id= device_id[1],
                active_modes = active_modes[1],
                TrackDataIdVar= TrackDataIdVar[1],
                mode3 = mode3[1],
                n_datapoints = n(),
                mean_accuracy= mean(accuracy),
                ratio_interpolated = sum(interpolated)/n(),
                ratio_filtered     = sum(filter1, na.rm=TRUE) / n(),
                mid_point = as_datetime(paste0("1970-01-01 ", substr(as.character(as.POSIXct(as.numeric(start_time[1] + as.numeric(end_time[2]))/2, origin = "1970-01-01")), 12, 19))),
                during_rush = mid_point %between% c(morning_start_rush, morning_end_rush) | mid_point %between% c(evening_start_rush, evening_end_rush)) 
    
    data <- bind_cols(c(data_needed,
                        "distance_straight_line"= distance_straight_line,
                        total_distance.segments,
                        total_distance,
                        "ratio_straightline_distance" = ratio_straightline_distance,
                        average_Speed_track,
                        speed_stats,
                        #angle_stats,
                        stat_urbanizity,
                        stat_pop_dens,
                        stats_nearby_station,
                        stats_nearby_train_tracks,
                        stats_nearby_highways,
                        stats_nearby_onofframps,
                        stats_nearby_subway_station,
                        stats_nearby_tram_stop,
                        stats_nearby_tram_tracks,
                        stats_nearby_subway_tracks,
                        stats_nearby_bus_stops))
    data_list[[i]] <- data
    saveRDS(data, paste0("./output_files/filtered_merged_tracks_stats/analyzed_",i, ".RDS"))
    print(paste0("finished:", i))
    Sys.sleep(.001)
  }, error= function(e){cat("ERROR :", conditionMessage(e), "\n")})
}
```

Once this list of single row data frames (1 per trip has been calculated), we can merge them into one master data set and save this one.

``` r
all_trips <- rbindlist(data_stats, fill= TRUE)
saveRDS(all_trips, "output_files/all_trips.RDS")
all_trips <- readRDS("output_files/all_trips.RDS")
```

calculate statistics of features
--------------------------------

Make plots
----------

This code was used to create the different plots of both the data and the results

``` r
# 1) load in the polygon of the Netherlands
data("NLD_muni")
NL <- st_union(NLD_muni)
NL2 <- sf::st_transform(NL, "+proj=longlat +datum=WGS84 +no_defs +ellps=WGS84 +towgs84=0,0,0")

# 2) load in the full data
locationswithtransport_full <- readRDS("output_files/locationswithtransport_full.RDS")

setDT(locationswithtransport_full)
locationswithtransport_full_unique <- unique(locationswithtransport_full, by = c("device_id", "TrackDataIdVar", "mode3",  "latitude", "longitude", "timestamp_location"))
rm(locationswithtransport_full)



# 3) plot the tracks
plot_all_tracks <- ggplot()+
  geom_path(data=locationswithtransport_full_unique, 
            aes(x =longitude, 
                y= latitude, 
                group= TrackDataIdVar),
            alpha=.4,
            col="grey40",
            size=.2)+
  geom_sf(data = NL2, fill=NA, col = "darkred",  size=0.3)+ 
  theme_map()+
  xlim(3.34, 7.2)+
  ylim(50.7,53.6)+
  theme(legend.position = "none",
        panel.grid.major = element_line(colour = 'transparent'))

# The file is too big to calculate a 2d density so a sample was taken
locationswithtransport_sample_unique <- locationswithtransport_full_unique %>%
  sample_frac(.5)

#4 ) plot the the heatmap
plot_all_heatmap <- ggplot()+
  stat_density2d(data=locationswithtransport_sample_unique, 
                 aes(x =longitude, 
                     y= latitude,
                     fill= ..level..,
                     alpha = ..level..),
                 geom  = "polygon",
                 size=5, 
                 bins=75,
                 n=150)+
  scale_fill_gradient( low = "yellow", high = "red", guide=FALSE)+
  scale_alpha(range=c(0.1, .9))+
  geom_sf(data = NL2, fill=NA, col = "darkred",  size=0.3)+ 
  theme_map()+
  xlim(3.34, 7.2)+
  ylim(50.7,53.6)+
  theme(legend.position = "none",
        panel.grid.major = element_line(colour = 'transparent'))

# we can remove this sample again after saving if we want
#rm(locationswithtransport_sample_unique)

# to get a working clock, we need to add the right time to the right frame
locationswithtransport_full_unique <- locationswithtransport_full_unique %>% 
  group_by(device_id )%>%
  arrange(timestamp_location)%>%
  mutate(time_hours =  strftime(timestamp_location, format= "%H:%M:%S"))%>%
  mutate(time_hours2= as.POSIXct(time_hours,format= "%H:%M:%S"))%>%
  mutate(clock =  round_date(timestamp_location, "5 mins")) %>%
  mutate(clock =  strftime(clock, format= "%H:%M"))

# 5) we can also animate all the trips in the data set
ani_tracks_sample2 <- ggplot()+
 geom_point(data=locationswithtransport_full_unique, 
             aes(x =longitude, 
                 y= latitude),
             alpha=.2,
             col="grey40",
             size=.7,
             shape= 20)+
  xlim(3.34, 7.2)+
  ylim(50.7,53.6)+
  geom_text(data=locationswithtransport_full_unique, x=4, y= 53, aes(label=clock), size= 20)+
  transition_time(time_hours2)+
  geom_sf(data = NL2, fill=NA, col = "darkred",  size=1)+ 
  shadow_wake(wake_length = .01, size= NULL, alpha= NULL, falloff= "linear", wrap= FALSE, exclude_layer = 2)+
  theme_map()+
  theme(legend.position = "none",
        panel.grid.major = element_line(colour = 'transparent'))

# 6 now we can save all the plots and animation
ggsave("plots/plot_all_tracks.png", plot_all_tracks, width = 10, units= "cm")
ggsave("plots/plot_all_tracks3_big.png", plot_all_tracks, width = 20, units= "cm")
ggsave("plots/plot_all_heatmap2.png", plot_all_heatmap, height =  25, units= "cm")
anim_save("plots/ani_tracks_sample26.gif", ani_tracks_sample2, nframes = 1200 , fps= 30, width = 2500, height =  2500, units= "px")
```

### some extra plots just for presentation purposes

These 3 plots are just for the presentation, but give an example of the type of features that will be used

``` r
long_format_statistics2 <- long_format_statistics %>%
  mutate(mode3 = ifelse(mode3 == "BikeNonElectric", "Bicycle",
                        ifelse(mode3 ==  "BikeElectric", " E-bike", 
                        ifelse(mode3 == "Walk", "Walking", 
                               ifelse(mode3 == "PublicBus", "Bus", mode3)))))


speed_plot <- long_format_statistics2 %>%
  filter(statistic == "median_speed")%>%
  ggplot()+
  geom_boxplot(aes(value, x= mode3,fill= mode3), alpha =  .8, outlier.alpha = .2)+
  ylim(0, 75)+
  scale_fill_viridis(discrete = TRUE)+
  theme_minimal()+
  theme(legend.position = "none", axis.text.x =  element_text(angle= 45))+
  labs(x= "", y="m/s", title = "Median speed per trip", fill = "transport mode")


subway_plot <- long_format_statistics2 %>%
  filter(statistic == "gpsPoints_nearby_subwaystation_start_and_end")%>%
  group_by(mode3) %>%
  ggplot()+
  geom_bar(aes(x=as.factor(value), group= as.factor(mode3), fill= mode3,  y = ..prop..),
           position = position_dodge(),  alpha =  .8)+
  scale_fill_viridis(discrete = TRUE)+
  theme_minimal()+
  
  labs(x= "", y="transport mode proportion", title = "Start & end of trip near to metro station", fill = "transport mode")
  scale_x_discrete(labels =  c("not near", "near"))+
  


bus_plot <- long_format_statistics2 %>%
  filter(statistic == "median_distance_bus_stops")%>%
  ggplot()+
  geom_boxplot(aes(value, x= mode3,  fill= mode3), alpha =  .8, outlier.alpha = .2)+
  ylim(0, 15)+
  scale_fill_viridis(discrete = TRUE)+
  theme_minimal()+
  theme(legend.position = "none", axis.text.x =  element_text(angle= 45))+
  labs(x= "", y="meter", title = "Median distance to bus stop" )


ggsave("plots/speed_plot.png", speed_plot, width = 5, height = 8)
ggsave("plots/subway_plot.png", subway_plot, width = 5,  height = 8)
ggsave("plots/bus_plot.png", bus_plot, width = 5,  height = 8)
```

Random Forest implementation
----------------------------

Once all the tracks are analysed, they can used as input to the Random Forest model. However, before the different respondent level statistics need to be merged to the trips. Also, we need to merge the interpolated and non-interpolated stats

### merge respondent level stats

These are just for the labelled trips.

``` r
analyzed_files <- list.files("./output_files/filtered_merged_tracks_stats")

analyzed_files_list <- list()
for(i in analyzed_files) {
  analyzed_files_list[[i]] <-
    readRDS(paste0("./output_files/filtered_merged_tracks_stats/", i))
}

analyzed_files_df <- rbindlist(analyzed_files_list, fill = TRUE)

aggdata <- read_sav("./external_imput_data/user_info/AVA-EVA Analysis data set - complete.sav")
setDT(aggdata)

# Changing variable names
setnames(aggdata, "LFT", "Age_in_yrs")
setnames(aggdata, "AUTOP", "Has_car")
setnames(aggdata, "BROMMEROP", "Has_moped")
setnames(aggdata, "RIJBEWIJSP", "Driverslicense")
setnames(aggdata, "STEDBUURT", "Urbanity")
setnames(aggdata, "INHP100HGEST", "Income")
setnames(aggdata, "INHP100HBEST", "Income2")

aggdata2 <- aggdata[, c("username",
                        "Geslacht" ,  
                        "Age_in_yrs",  
                        "Has_car", 
                        "Urbanity", 
                        "Income", 
                        "Income2", 
                        "Has_moped" ,
                        "Driverslicense", 
                        "LEASEAUTO",
                        "STEDGEM",
                        "INPSECJ" )]


aggdata2 <- aggdata2 %>%
  as_tibble() %>%
  mutate(username = as.character(username))

# 1) list all the folders in the Imput_data folder
folders<-list.dirs(path ="./Imput_data",
                   full.names=F,
                   recursive = F)

# 2) create an empty list to store the transport modes in
users_list <- list()

for(i in 1:length(folders)){
  users_list[[i]]<-fread(paste0(getwd(), "/Imput_data/",
                                folders[i], 
                                "/users.csv"))
}


users_1101_0220 <-rbindlist(users_list) %>%
  select(id, username) %>%
  distinct()%>%
  rename("user_id" = id)


device_list <- list()

for(i in 1:length(folders)){
  device_list[[i]]<-fread(paste0(getwd(), "/Imput_data/",
                                 folders[i], 
                                 "/devices.csv"))
}


device__1101_0220 <-rbindlist(device_list) %>%
  select(-created_at, -updated_at, -deleted_at) %>%
  distinct() %>%
  rename("device_id" = id)

joined_user_device_info <- left_join(device__1101_0220, users_1101_0220, by = "user_id")
setDT(joined_user_device_info)


all_user_info <- left_join(joined_user_device_info, aggdata2, by = "username")

setDT(all_user_info)
setDT(analyzed_files_df)


all_trips_with_person_char <- left_join(analyzed_files_df, all_user_info, by = "device_id")
```

### merging data frames

After collecting the relevant respondent level statistics, they can be merged. In this step also some missingness in the data is solved. For example some divisions by 0 occurred in the code, that are handled here. Due to the radius set above, it did occur that the distance to different public transport infrastructures was over 40km and therefore completely missing. As mentioned in the the thesis, 3 different levels of features are distinguished, the names of features are put in different vectors here.

``` r
analyzed_files_intrapolated <- list.files("./output_files/filtered_merged_tracks_stats3")


analyzed_files_list_intrapolated <- list()
for(i in analyzed_files_intrapolated) {
  analyzed_files_list_intrapolated[[i]] <-
    readRDS(paste0("./output_files/filtered_merged_tracks_stats3/", i))
}

analyzed_files_df_intrapolated <- rbindlist(analyzed_files_list_intrapolated, fill = TRUE)



# these are omitted, because they do not add do not make sense as sensical features
analyzed_files_df_intrapolated <- analyzed_files_df_intrapolated %>%
  select(-mean_dist_betweenpoint,
         -mean_turn,
         -median_turn,
         -n_turn_1plus90,
         -n_turn_1plus150,
         -n_turn_5plus90,
         -n_turn_5plus150,
         -turn_1plus90_ps,
         -turn_1plus150_ps,
         -turn_5plus90_ps,
         -turn_1plus150_ps)

for(i in 1:ncol(analyzed_files_df_intrapolated)){
  names(analyzed_files_df_intrapolated)[i] <- paste0("intrapolated_",names(analyzed_files_df_intrapolated)[i])
}

vars_to_delete <- c("os_version", "username",  "user_id", "manufacturer", "model", "os", "device_id" , "active_modes", "TrackDataIdVar")
vars_level1    <- c("mid_point", "n_segments", "n_datapoints",  "distance_straight_line","total_distance.segments" ,
                    "total_distance", "average_Speed_track","mean_bearing","median_bearing" ,                             
                    "max_bearing", "bearing_0.05" , "bearing_0.25" ,                               
                    "bearing_0.75", "bearing_0.9", "bearing_0.95",                               
                    "bearing_0.99" ,"bearing_sd"  ,"mean_abs.bearing",                            
                    "median_abs.bearing","max_abs.bearing","abs.bearing_0.05",                           
                    "abs.bearing_0.25","abs.bearing_0.75" ,"abs.bearing_0.9",                          
                    "abs.bearing_0.95" ,"abs.bearing_0.99","abs.bearing_sd",                              
                    "mean_rel_bearing" ,  "median_rel_bearing","max_rel_bearing" ,                           
                    "rel_bearing_0.05" ,"rel_bearing_0.25","rel_bearing_0.75",                          
                    "rel_bearing0.9","rel_bearing_0.95","rel_bearing_0.99",                            
                    "rel_bearing_sd", "mean_speed","median_speed",                                
                    "max_speed","speed_0.05","speed_0.25",                                  
                    "speed_0.75" ,"speed_0.9","speed_0.95",                                  
                    "speed_0.99", "speed_sd" ,"speed_skew",   
                    "speed_above80_ratio" ,"speed_above120_ratio", "speed_below5_ratio",                          
                    "n_inf_acc","mean_acceleration","median_acceleration",                         
                    "max_acceleration","min_acceleration","acceleration_0.01" ,                          
                    "acceleration_0.05","acceleration_0.1","acceleration_0.25",                           
                    "acceleration_0.75","acceleration_0.9","acceleration_0.95",                           
                    "acceleration_0.99","acceleration_sd","acceleration_skew" ,                          
                    "mean_dist_betweenpoint","mean_turn","median_turn",                                 
                    "n_turn_1plus90","n_turn_1plus150","n_turn_5plus90" ,                             
                    "n_turn_5plus150","turn_1plus90_ps","turn_1plus150_ps",                            
                    "turn_5plus90_ps","turn_5plus150_ps", "ratio_straightline_distance.total_distance","diff_time_total")
vars_level2    <- c("n_urban_area","unique_urban_area", "ratio_urban_area" , "mean_bev_dens" ,"median_bev_dens" ,"mean_urban" , "sdBEV", "median_urban" ,  "mode_urban" ,
                    "ratio_in_urban","gpsPoints_nearby_train_station"  , "mean_distance_train_station","median_distance_train_station",  "gpsPoints_nearby_train_station_start",
                    "gpsPoints_nearby_train_station_end" ,"gpsPoints_nearby_station_start_and_end","unique_train_stations", "near_trainstation_relative1","near_trainstation_relative2",
                    "gpsPoints_nearby_train_track" ,"mean_distance_traintracks", "median_distance_traintracks" ,"near_traintracks_relative1","near_traintracks_relative2", 
                    "gpsPoints_nearby_highways" ,"mean_distance_highways","median_distance_highways" , "near_highways_relative1","near_highways_relative2", "gpsPoints_nearby_onofframps" ,
                    "mean_distance_onofframps", "median_distance_onofframps" ,"unique_train_onofframps", "near_onofframps_relative1" ,"near_onofframps_relative2", 
                    "gpsPoints_nearby_subway_station","mean_distance_subway_station" , "median_distance_subway_station","gpsPoints_nearby_subway_station_start",
                    "gpsPoints_nearby_subway_station_end", "gpsPoints_nearby_subwaystation_start_and_end", "unique_subway_stations","near_subway_station_relative1" , 
                    "near_subway_station_relative2","gpsPoints_nearby_tram_stop", "mean_distance_tram_stop","median_distance_tram_stop" ,"gpsPoints_nearby_tram_stop_start",
                    "gpsPoints_nearby_tram_stop_end", "gpsPoints_nearby_tram_stop_start_and_end" ,"unique_tram_stop"  , "near_tram_stop_relative1" ,"near_tram_stop_relative2" ,
                    "gpsPoints_nearby_tram_track", "mean_distance_tram_track" , "median_distance_tram_track", "near_tramtracks_relative1","near_tramtracks_relative2" , 
                    "gpsPoints_nearby_subway_track" , "mean_distance_subway_track","median_distance_subway_track" , "near_subwaytrack_relative1","near_subwaytrack_relative2",
                    "gpsPoints_nearby_bus_stops", "mean_distance_bus_stops","median_distance_bus_stops", "gpsPoints_nearby_bus_stops_start",            
                    "gpsPoints_nearby_bus_stops_end" , "gpsPoints_nearby_bus_stops_start_and_end", "unique_bus_stops" ,                           
                    "near_bus_stops_relative1" ,"near_bus_stops_relative2","during_rush", "mean_accuracy" ,"ratio_filtered" )
vars_level3    <- c( "Geslacht","Age_in_yrs","Has_car", "Urbanity", "Income","Income2", "Has_moped","Driverslicense" ,"LEASEAUTO","INPSECJ", "STEDGEM" )



vars_to_delete_dummy <- c(paste0("dummyNA_",vars_to_delete),"dummyNA_mode3")
vars_level1_dummy <- paste0("dummyNA_",vars_level1)
vars_level2_dummy <- paste0("dummyNA_",vars_level2)
vars_level3_dummy <- paste0("dummyNA_",vars_level3)
vars_to_delete_dummy <- vars_to_delete_dummy[vars_to_delete_dummy!="dummyNA_TrackDataIdVar"]
vars_to_delete_intrapolated <- c(paste0("intrapolated_",vars_to_delete), "intrapolated_active_modes", "intrapolated_mode3", "intrapolated_TrackDataIdVar")
vars_level1_intrapolated <- paste0("intrapolated_",vars_level1)
vars_level2_intrapolated <- paste0("intrapolated_",vars_level2)
vars_level3_intrapolated <- paste0("intrapolated_",vars_level3)


matrix_dummies <- as.data.frame(matrix(NA, ncol= ncol(all_trips_with_person_char), nrow= nrow(all_trips_with_person_char)))

for(i in 1:ncol(all_trips_with_person_char)){
  var<-all_trips_with_person_char[,i]
  matrix_dummies[,i] <- is.na(var)
  names(matrix_dummies)[i] <- paste0("dummyNA_",names(all_trips_with_person_char)[i])
}


all_trips_with_person_char_missing_fixed <- all_trips_with_person_char %>%
  mutate(ratio_straightline_distance.total_distance = ifelse(is.na(ratio_straightline_distance.total_distance),1, ratio_straightline_distance.total_distance))%>%
  mutate(average_Speed_track = ifelse(is.na(average_Speed_track), 0, average_Speed_track))%>%
  mutate(median_speed = ifelse(is.na(median_speed), 0, median_speed))%>%
  mutate(mean_speed = ifelse(is.na(mean_speed), 0, mean_speed))%>%
  mutate_if(grepl("_acceleration", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("bearing", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("_relative1", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("max_speed", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("acceleration_", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_acceleration", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("bearing", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("speed_", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_speed", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_sd", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("mean_distance_", names(.)), ~if_else(is.na(.), 150, .))%>%
  mutate_if(grepl("median_distance_", names(.)), ~if_else(is.na(.), 150, .)) %>%
  mutate_if(grepl("_relative1", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate_if(grepl("_ps", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate_if(grepl("mean_dist_betweenpoint", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate(mean_bev_dens = ifelse(is.na(mean_bev_dens), median(all_trips_with_person_char$mean_bev_dens, na.rm=TRUE), mean_bev_dens))%>%
  mutate(median_bev_dens = ifelse(is.na(median_bev_dens), median(all_trips_with_person_char$median_bev_dens, na.rm=TRUE), median_bev_dens))%>%
  mutate(sdBEV = ifelse(is.na(sdBEV), median(all_trips_with_person_char$sdBEV, na.rm=TRUE), sdBEV))%>%
  mutate(median_urban = ifelse(is.na(median_urban), median(all_trips_with_person_char$median_urban, na.rm=TRUE), median_urban))%>%
  mutate(mode_urban = ifelse(is.na(mode_urban), median(all_trips_with_person_char$mode_urban, na.rm=TRUE), mode_urban))%>%
  mutate(ratio_in_urban = ifelse(is.na(ratio_in_urban), median(all_trips_with_person_char$ratio_in_urban, na.rm=TRUE), ratio_in_urban))%>%
  mutate(mean_urban = ifelse(is.na(mean_urban), median(all_trips_with_person_char$mean_urban, na.rm=TRUE), mean_urban))%>%
  mutate(Age_in_yrs = as.numeric(Age_in_yrs))%>%
  mutate(Has_car = as.numeric(Has_car))%>%
  mutate(Urbanity = as.numeric(Urbanity))%>%
  mutate(Income2 = as.numeric(Income2))%>%
  mutate(Income = as.numeric(Income))%>%
  mutate(LEASEAUTO = as.numeric(LEASEAUTO))%>%
  mutate(Driverslicense = as.numeric(Driverslicense))%>%
  mutate(STEDGEM = as.numeric(STEDGEM))%>%
  mutate(INPSECJ = as.numeric(INPSECJ))%>%
  mutate(Has_moped = as.numeric(Has_moped))

analyzed_files_df_intrapolated_missing_fixed <- analyzed_files_df_intrapolated %>%
  mutate(intrapolated_ratio_straightline_distance.total_distance = ifelse(is.na(intrapolated_ratio_straightline_distance.total_distance),1, intrapolated_ratio_straightline_distance.total_distance))%>%
  mutate(intrapolated_average_Speed_track = ifelse(is.na(intrapolated_average_Speed_track), 0, intrapolated_average_Speed_track))%>%
  mutate(intrapolated_median_speed = ifelse(is.na(intrapolated_median_speed), 0, intrapolated_median_speed))%>%
  mutate(intrapolated_mean_speed = ifelse(is.na(intrapolated_mean_speed), 0, intrapolated_mean_speed))%>%
  mutate_if(grepl("_acceleration", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("bearing", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("_relative1", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("max_speed", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("acceleration_", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_acceleration", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("bearing", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("speed_", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_speed", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_sd", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("mean_distance_", names(.)), ~if_else(is.na(.), 150, .))%>%
  mutate_if(grepl("median_distance_", names(.)), ~if_else(is.na(.), 150, .)) %>%
  mutate_if(grepl("_relative1", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate_if(grepl("_ps", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate_if(grepl("mean_dist_betweenpoint", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate(intrapolated_mean_bev_dens = ifelse(is.na(intrapolated_mean_bev_dens), median(analyzed_files_df_intrapolated$intrapolated_mean_bev_dens, na.rm=TRUE), intrapolated_mean_bev_dens))%>%
  mutate(intrapolated_median_bev_dens = ifelse(is.na(intrapolated_median_bev_dens), median(analyzed_files_df_intrapolated$intrapolated_median_bev_dens, na.rm=TRUE), intrapolated_median_bev_dens))%>%
  mutate(intrapolated_sdBEV = ifelse(is.na(intrapolated_sdBEV), median(intrapolated_analyzed_files_df_intrapolated$intrapolated_sdBEV, na.rm=TRUE), intrapolated_sdBEV))%>%
  mutate(intrapolated_median_urban = ifelse(is.na(intrapolated_median_urban), median(analyzed_files_df_intrapolated$intrapolated_median_urban, na.rm=TRUE), intrapolated_median_urban))%>%
  mutate(intrapolated_mode_urban = ifelse(is.na(intrapolated_mode_urban), median(analyzed_files_df_intrapolated$intrapolated_mode_urban, na.rm=TRUE), intrapolated_mode_urban))%>%
  mutate(intrapolated_ratio_in_urban = ifelse(is.na(intrapolated_ratio_in_urban), median(analyzed_files_df_intrapolated$intrapolated_ratio_in_urban, na.rm=TRUE), intrapolated_ratio_in_urban))%>%
  mutate(intrapolated_mean_urban = ifelse(is.na(intrapolated_mean_urban), median(analyzed_files_df_intrapolated$intrapolated_mean_urban, na.rm=TRUE), intrapolated_mean_urban))

all_trips_with_person_char_missing_fixed_with_dummies <- bind_cols(all_trips_with_person_char_missing_fixed, matrix_dummies)


all_trips_with_person_char_missing_fixed_with_dummies_and_intrap <- bind_cols(all_trips_with_person_char_missing_fixed_with_dummies,
                                                                              analyzed_files_df_intrapolated_missing_fixed)
```

### run different models

The thesis separates nine different models, the code for all of these is presented here. not all the code for all the presented models is listed (for example the code to run the third model without user errors). However, with the present code, these models can easily be created.

``` r
tuneLength_set <- 100 # how many iterations of hyper parameter tuning 

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -vars_level2_dummy,
         -vars_level3_dummy, 
         -vars_level2, 
         -vars_level3,
         -vars_level2_intrapolated,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar) %>%
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3))  # this is needed, because no spaces are allowed in the algorithm 

names(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1)

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1_PP <- preProcess(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1, method= "medianImpute")
all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1 <- predict(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1_PP, all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1)

anyNA(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1) # this is to check for completeness of the data (a requirement for the Random Forest.)

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1_sub5 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1 %>%
  filter(n_datapoints > 2)

set.seed(1)
control_rf <- trainControl(method = "repeatedcv", 
                           number = 5,
                           repeats = 10,
                           search = "random",
                           savePredictions = "final", 
                           classProbs = TRUE,
                           sampling = "down")


rf_output_vars1<- train(mode3~., 
                        data=all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1_sub5, 
                        method="ranger",
                        tuneLength=tuneLength_set,
                        trControl=control_rf, 
                        num.threads=6,
                        metric= "Kappa",
                        importance ='permutation')

###################

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars12 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -vars_level3_dummy, 
         -vars_level3,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar) %>%
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3))  # this is needed, because no spaces are allowed in the algorithm 


all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars12_PP <- preProcess(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars12,
                                                                                         method= "medianImpute")
all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars12 <- predict(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars12_PP, 
                                                                                   all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars12)


all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars12_sub5 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars12 %>%
  filter(n_datapoints > 2)

set.seed(1)
control_rf <- trainControl(method = "repeatedcv", 
                           number = 5,
                           repeats = 10,
                           search = "random",
                           savePredictions = "final", 
                           classProbs = TRUE,
                           sampling = "down")


rf_output_vars12<- train(mode3~., 
                         data = all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars12_sub5, 
                         method = "ranger",
                         tuneLength = tuneLength_set,
                         trControl = control_rf, 
                         num.threads = 6,
                         metric = "Kappa",
                         importance ='permutation')

###################

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar) %>%
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3))  # this is needed, because no spaces are allowed in the algorithm 


all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_PP <- preProcess(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123,
                                                                                          method= "medianImpute")
all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123 <- predict(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_PP, 
                                                                                    all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123)


all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123 %>%
  filter(n_datapoints > 2)

set.seed(1)
control_rf <- trainControl(method = "repeatedcv", 
                           number = 5,
                           repeats = 10,
                           search = "random",
                           savePredictions = "final", 
                           classProbs = TRUE,
                           sampling = "down")


rf_output_vars123<- train(mode3~., 
                          data = all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5, 
                          method = "ranger",
                          tuneLength = tuneLength_set,
                          trControl = control_rf, 
                          num.threads = 6,
                          metric= "Kappa",
                          importance ='permutation')


###################


all_trips_with_person_char_missing_fixed_with_dummies_combinedbikes_modes_vars123 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar) %>%
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3)) %>%
  mutate(mode3 = ifelse(mode3 %in% c("BikeNonElectric","BikeElectric"), "Bikecombined", mode3))


all_trips_with_person_char_missing_fixed_with_dummies_combinedbikes_modes_vars123_PP <- preProcess(all_trips_with_person_char_missing_fixed_with_dummies_combinedbikes_modes_vars123, method= "medianImpute")
all_trips_with_person_char_missing_fixed_with_dummies_combinedbikes_modes_vars123 <- predict(all_trips_with_person_char_missing_fixed_with_dummies_combinedbikes_modes_vars123_PP, 
                                                                                             all_trips_with_person_char_missing_fixed_with_dummies_combinedbikes_modes_vars123)

all_trips_with_person_char_missing_fixed_with_dummies_combinedbikes_modes_vars123_sub5 <- all_trips_with_person_char_missing_fixed_with_dummies_combinedbikes_modes_vars123 %>%
  filter(n_datapoints > 2)

set.seed(1)
control_rf <- trainControl(method = "repeatedcv", 
                           number = 5,
                           repeats = 10,
                           search = "random",
                           savePredictions = "final", 
                           classProbs = TRUE,
                           sampling = "down")


rf_output_vars123_combinedbikes <- train(mode3~., 
                                         data=all_trips_with_person_char_missing_fixed_with_dummies_combinedbikes_modes_vars123_sub5, 
                                         method="ranger",
                                         tuneLength=tuneLength_set,
                                         trControl=control_rf, 
                                         num.threads=6,
                                         metric= "Kappa",
                                         importance ='permutation')



###################


all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motor_modes_vars123 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar) %>%
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3)) %>%
  mutate(mode3 = ifelse(mode3 %in% c("BikeNonElectric","BikeElectric"), "Bikecombined", mode3))%>%
 mutate(mode3 = ifelse(mode3 %in% c("Car", "Scooter"), "Motorized", mode3))


 all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motor_modes_vars123_PP <- preProcess(all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motor_modes_vars123, method= "medianImpute")
 all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motor_modes_vars123 <- predict(all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motor_modes_vars123_PP, 
                                                                                                   all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motor_modes_vars123)

all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motor_modes_vars123_sub5 <- all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motor_modes_vars123 %>%
  filter(n_datapoints > 2)

set.seed(1)
control_rf <- trainControl(method = "repeatedcv", 
                           number = 5,
                           repeats = 10,
                           search = "random",
                           savePredictions = "final", 
                           classProbs = TRUE,
                           sampling = "down")


rf_output_vars123_combinedbikesmotor <- train(mode3~., 
                                         data=all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motor_modes_vars123_sub5, 
                                         method="ranger",
                                         tuneLength=tuneLength_set,
                                         trControl=control_rf, 
                                         num.threads=6,
                                         metric= "Kappa",
                                         importance ='permutation')


###################


all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic1_modes_vars123 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar) %>%
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3)) %>%
  mutate(mode3 = ifelse(mode3 %in% c("BikeNonElectric","BikeElectric"), "Bikecombined", mode3))%>%
mutate(mode3 = ifelse(mode3 %in% c("Metro", "Tram"), "Publiccombined", mode3)) %>%
mutate(mode3 = ifelse(mode3 %in% c("Car", "Scooter"), "Motorized", mode3))


all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic1_modes_vars123_PP <- preProcess(all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic1_modes_vars123,  method= "medianImpute")

all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic1_modes_vars123 <- predict(all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic1_modes_vars123_PP, all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic1_modes_vars123)

all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic1_modes_vars123_sub5 <- all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic1_modes_vars123 %>%
  filter(n_datapoints > 2)

set.seed(1)
control_rf <- trainControl(method = "repeatedcv", 
                           number = 5,
                           repeats = 10,
                           search = "random",
                           savePredictions = "final", 
                           classProbs = TRUE,
                           sampling = "down")


rf_output_vars123_combinedbikesmotorpublic1 <- train(mode3~., 
                                              data = all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic1_modes_vars123_sub5, 
                                              method="ranger",
                                              tuneLength=tuneLength_set,
                                              trControl=control_rf, 
                                              num.threads=6,
                                              metric= "Kappa",
                                              importance ='permutation')





###################

all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar) %>%
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3)) %>%
  mutate(mode3 = ifelse(mode3 %in% c("BikeNonElectric","BikeElectric"), "Bikecombined", mode3))%>%
  mutate(mode3 = ifelse(mode3 %in% c("Metro", "Tram", "PublicBus", "Train"), "Publiccombined", mode3)) %>%
  mutate(mode3 = ifelse(mode3 %in% c("Car", "Scooter"), "Motorized", mode3))


all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123_PP <- preProcess(all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123,  method= "medianImpute")
all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123 <- predict(all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123_PP, 
                                                                                                         all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123)

anyNA(all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123)
all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123_sub5 <- all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123 %>%
  filter(n_datapoints > 2)

set.seed(1)
control_rf <- trainControl(method ="repeatedcv", 
                           number = 5,
                           repeats = 10,
                           search = "random",
                           savePredictions = "final", 
                           classProbs = TRUE,
                           sampling = "down")


rf_output_vars123_combinedbikesmotorpublic3 <- train(mode3~.,  data=all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic3_modes_vars123_sub5, 
                                                     method="ranger",
                                                     tuneLength=tuneLength_set,
                                                     trControl=control_rf, 
                                                     num.threads=6,
                                                     metric= "Kappa",
                                                     importance ='permutation')
###################

all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic4_modes_vars123 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar) %>%
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3)) %>%
  mutate(mode3 = ifelse(mode3 %in% c("BikeNonElectric","BikeElectric"), "Bikecombined", mode3))%>%
  mutate(mode3 = ifelse(mode3 %in% c("Metro", "Tram", "Train"), "Publiccombined", mode3)) %>%
  mutate(mode3 = ifelse(mode3 %in% c("Car", "Scooter"), "Motorized", mode3))


all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic4_modes_vars123_PP <- preProcess(all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic4_modes_vars123, method= "medianImpute")
all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic4_modes_vars123 <- predict(all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic4_modes_vars123_PP, 
                                                                                                         all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic4_modes_vars123)

all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic4_modes_vars123_sub5 <- all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic4_modes_vars123 %>%
  filter(n_datapoints > 2)

set.seed(1)
control_rf <- trainControl(method="repeatedcv", 
                           number=5,
                           repeats = 10,
                           search = "random",
                           savePredictions = "final", 
                           classProbs = TRUE,
                           sampling = "down")


rf_output_vars123_combinedbikesmotorpublic4 <- train(mode3~., 
                                                     data = all_trips_with_person_char_missing_fixed_with_dummies_combinedbike_motorpublic4_modes_vars123_sub5, 
                                                     method="ranger",
                                                     tuneLength=tuneLength_set,
                                                     trControl=control_rf, 
                                                     num.threads=6,
                                                     metric= "Kappa",
                                                     importance ='permutation')

###################


all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar,
         -mid_point,
         -intrapolated_mid_point)  %>%
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi"&
           mode3 != "User error") %>%
  mutate(mode3 = ifelse(mode3 %in% c("BikeNonElectric","BikeElectric"), "Bikecombined", mode3))%>%
  mutate(mode3 = ifelse(mode3 %in% c("PublicBus","Metro", "Tram", "Train"), "Publiccombined", mode3)) %>%
  mutate(mode3 = ifelse(mode3 %in% c("Car", "Scooter"), "Motorized", mode3))

all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123_PP <- preProcess(all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123,
                                                                                                          method= "medianImpute")
all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123 <- predict(all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123_PP, 
                                                                                                    all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123)

all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123_sub5 <- all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123 %>%
  filter(n_datapoints > 2)

set.seed(1)
control_rf <- trainControl(method="repeatedcv", 
                           number=5,
                           repeats = 10,
                           search = "random",
                           savePredictions = "final", 
                           classProbs = TRUE,
                           sampling = "down")


rf_output_vars123_combinedpublic_noer <- train(mode3~., 
                                               data = all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123_sub5, 
                                               method="ranger",
                                               tuneLength=tuneLength_set,
                                               trControl=control_rf, 
                                               num.threads=6,
                                               metric= "Kappa",
                                               importance ='permutation')
```

model with dummies or proportions of earlier used transport modes
-----------------------------------------------------------------

This code can be used to include the proportions of earlier used transport modes. To chance between the two, please comment out the necessary lines of code

``` r
test_set <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap %>%
  group_by(device_id, mode3)%>%
  summarise(rr = n())%>%
  arrange(device_id)%>%
  ungroup()%>%
  spread(mode3, rr, fill = 0)%>%
  rename_at(vars(-device_id), function(x) paste0(x, "_used"))

#saveRDS(test_set, "test_set.RDS")


all_trips_with_person_char_missing_fixed_with_dummies_and_intrap2 <- left_join(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap, test_set)

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap2 %>%
  mutate(total_trips = BikeElectric_used+BikeNonElectric_used+Car_used+Lorry_used+Metro_used+Motorbike_used+ multi_used+ other_transport_mode_used+ PublicBus_used+Train_used+
           `trip saved twice with multi modes_used`+ `User error_used`+ Walk_used) %>%
  mutate(BikeElectric_used = ifelse(mode3== "BikeElectric", BikeElectric_used-1, BikeElectric_used))%>%
  mutate(BikeNonElectric_used = ifelse(mode3== "BikeNonElectric", BikeNonElectric_used-1, BikeNonElectric_used))%>%
  mutate(Car_used = ifelse(mode3== "Car", Car_used-1, Car_used))%>%
  mutate(Lorry_used = ifelse(mode3== "Lorry", Lorry_used-1, Lorry_used))%>%
  mutate(Metro_used = ifelse(mode3== "Metro", Metro_used-1, Metro_used))%>%
  mutate(Motorbike_used = ifelse(mode3== "Motorbike", Motorbike_used-1, Motorbike_used))%>%
  mutate(multi_used = ifelse(mode3== "multi", multi_used-1, multi_used))%>%
  mutate(other_transport_mode_used = ifelse(mode3== "other_transport_mode", other_transport_mode_used-1, other_transport_mode_used))%>%
  mutate(PublicBus_used = ifelse(mode3== "PublicBus", PublicBus_used-1, PublicBus_used))%>%
  mutate(Scooter_used = ifelse(mode3== "Scooter", Scooter_used-1, Scooter_used))%>%
  mutate(Train_used = ifelse(mode3== "Train", Train_used-1, Train_used))%>%
  mutate(`trip saved twice with multi modes_used` = ifelse(mode3== "trip saved twice with multi modes_used", `trip saved twice with multi modes_used`-1, `trip saved twice with multi modes_used`))%>%
  mutate(`User error_used` = ifelse(mode3== "User error_used", `User error_used`-1, `User error_used`))%>%
  mutate(Walk_used = ifelse(mode3== " Walk",  Walk_used-1,  Walk_used))#%>%
   #mutate_at(vars(ends_with("_used")),function(x) ifelse(x==0, FALSE, TRUE))  # this part of the code can be use to create dummies instead, buut then leave out, last two lines. 
  #mutate_at(vars(ends_with("_used")),function(x) x/total_trips)
all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3[,500:514] <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3[,500:514]/all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3[,515]

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap<-all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3[,1:514] 
```

results
-------

These are all the confusion matrices, accuracies and F1 statistics.

``` r
mat1 <- confusionMatrix(rf_output_vars123.noerror$pred$pred, rf_output_vars1$pred$obs) 
mat2 <- confusionMatrix(rf_output_vars12$pred$pred, rf_output_vars12$pred$obs) 
mat3 <- confusionMatrix(rf_output_vars123$pred$pred, rf_output_vars123$pred$obs) ###
mat4 <- confusionMatrix(rf_output_vars123_combinedbikes$pred$pred, rf_output_vars123_combinedbikes$pred$obs) ###
mat5 <- confusionMatrix(rf_output_vars123_combinedbikesmotor$pred$pred, rf_output_vars123_combinedbikesmotor$pred$obs) ###
mat6 <- confusionMatrix(rf_output_vars123_combinedbikesmotorpublic1$pred$pred, rf_output_vars123_combinedbikesmotorpublic1$pred$obs) ###
mat7 <- confusionMatrix(rf_output_vars123_combinedbikesmotorpublic4$pred$pred, rf_output_vars123_combinedbikesmotorpublic4$pred$obs) ###
mat8 <- confusionMatrix(rf_output_vars123_combinedbikesmotorpublic3$pred$pred, rf_output_vars123_combinedbikesmotorpublic3$pred$obs) ###
mat9 <- confusionMatrix(rf_output_vars123_combinedpublic_noer$pred$pred, rf_output_vars123_combinedpublic_noer$pred$obs) ###
```

### Plotting results

In addition to reporting the results, these can be plotted too. To get the publication ready plots, some further polishing in Adobe Illustrator was used.

``` r
list_outcomes <- list(mat1, mat2, mat3, mat4, mat5, mat6, mat7, mat8, mat9)

accuracy <- c()
macro_F1 <- c()
Sensitivity_Car <- c()
Sensitivity_Metro <- c()
Sensitivity_Tram <- c()
Sensitivity_Bus <- c()
Sensitivity_BikeElectric <- c()
Sensitivity_BikeNonElectric <-c()
Sensitivity_Walk <- c()
Sensitivity_Train<- c()
Sensitivity_Scooter <-c()
Sensitivity_Bikecombined<- c()
Sensitivity_Publiccombined <- c()
Sensitivity_Motorized <-c()
Sensitivity_User_error<- c()

for(i in 1:length(list_outcomes)){
  mat <- list_outcomes[[i]]
  accuracy[i]<-mat$overall[1]
  macro_F1[i] <- mean(mat$byClass[, "F1"])
  Sensitivity_Car[i]  <- ifelse("Class: Car" %in% row.names(mat$byClass), mat$byClass["Class: Car","Sensitivity"], NA)
  Sensitivity_Metro[i] <- ifelse("Class: Metro" %in% row.names(mat$byClass), mat$byClass["Class: Metro","Sensitivity"], NA)
  Sensitivity_Tram[i] <- ifelse("Class: Tram" %in% row.names(mat$byClass), mat$byClass["Class: Tram","Sensitivity"], NA)
  Sensitivity_Bus[i] <- ifelse("Class: PublicBus" %in% row.names(mat$byClass), mat$byClass["Class: PublicBus","Sensitivity"], NA)
  Sensitivity_BikeElectric[i] <- ifelse("Class: BikeElectric" %in% row.names(mat$byClass), mat$byClass["Class: BikeElectric","Sensitivity"], NA)
  Sensitivity_BikeNonElectric[i] <- ifelse("Class: BikeNonElectric" %in% row.names(mat$byClass), mat$byClass["Class: BikeNonElectric","Sensitivity"], NA)
  Sensitivity_Walk[i] <-ifelse("Class: Walk" %in% row.names(mat$byClass), mat$byClass["Class: Walk","Sensitivity"], NA)
  Sensitivity_Train[i] <-ifelse("Class: Train" %in% row.names(mat$byClass), mat$byClass["Class: Train","Sensitivity"], NA)
  Sensitivity_Scooter[i] <-ifelse("Class: Scooter" %in% row.names(mat$byClass), mat$byClass["Class: Scooter","Sensitivity"], NA)
  Sensitivity_Bikecombined[i] <-ifelse("Class: Bikecombined" %in% row.names(mat$byClass), mat$byClass["Class: Bikecombined","Sensitivity"], NA)
  Sensitivity_Publiccombined[i] <-ifelse("Class: Publiccombined" %in% row.names(mat$byClass), mat$byClass["Class: Publiccombined","Sensitivity"], NA)
  Sensitivity_Motorized[i] <-ifelse("Class: Motorized" %in% row.names(mat$byClass), mat$byClass["Class: Motorized","Sensitivity"], NA)
  Sensitivity_User_error[i] <-ifelse("Class: User_error" %in% row.names(mat$byClass), mat$byClass["Class: User_error","Sensitivity"], NA)
  
}


models_summarized2 <- tibble(models   = 1:length(list_outcomes),
                             accuracy =  accuracy,
                             macro_F1 = macro_F1,
                             Sensitivity_Car =  Sensitivity_Car,
                             Sensitivity_Metro =  Sensitivity_Metro,
                             Sensitivity_Tram =  Sensitivity_Tram,
                             Sensitivity_Bus =  Sensitivity_Bus,
                             Sensitivity_BikeElectric =  Sensitivity_BikeElectric,
                             Sensitivity_BikeNonElectric =  Sensitivity_BikeNonElectric,
                             Sensitivity_Walk =  Sensitivity_Walk,
                             Sensitivity_Train =  Sensitivity_Train,
                             Sensitivity_Scooter =  Sensitivity_Scooter,
                             Sensitivity_Bikecombined =  Sensitivity_Bikecombined,
                             Sensitivity_Publiccombined =  Sensitivity_Publiccombined,
                             Sensitivity_Motorized =  Sensitivity_Motorized,
                             Sensitivity_User_error =Sensitivity_User_error) %>%
  mutate(Sensitivity_User_error= ifelse(models==6, Sensitivity_User_error+.008, Sensitivity_User_error))%>%
  mutate(Sensitivity_Publiccombined= ifelse(models==9, Sensitivity_Publiccombined-.005, Sensitivity_Publiccombined))%>%
  mutate(Sensitivity_Walk= ifelse(models==6, Sensitivity_Walk-.008, Sensitivity_Walk))%>%
  mutate(Sensitivity_Walk= ifelse(models==9, Sensitivity_Walk+.005, Sensitivity_Walk))%>%
  mutate(Sensitivity_Bikecombined= ifelse(models==9, Sensitivity_Bikecombined-.008, Sensitivity_Bikecombined))%>%
  # mutate(Sensitivity_Motorized= ifelse(models==5, Sensitivity_Motorized-.01, Sensitivity_Motorized))%>%
  # mutate(Sensitivity_Car= ifelse(models==5, Sensitivity_Car-.01, Sensitivity_Car))%>%
  # mutate(Sensitivity_Scooter= ifelse(models==5, Sensitivity_Scooter-.01, Sensitivity_Scooter))%>%
  # mutate(Sensitivity_Bikecombined= ifelse(models==6, Sensitivity_Bikecombined-.005, Sensitivity_Bikecombined))%>%
  # mutate(Sensitivity_User_error= ifelse(models==8, Sensitivity_User_error+.007, Sensitivity_User_error))%>%
  # mutate(Sensitivity_Train= ifelse(models==5, Sensitivity_Train+.002, Sensitivity_Train))%>%
  # mutate(Sensitivity_Scooter= ifelse(models==4, Sensitivity_Scooter-.007, Sensitivity_Scooter))%>%
  # mutate(Sensitivity_BikeNonElectric= ifelse(models==1, Sensitivity_BikeNonElectric-.003, Sensitivity_BikeNonElectric))%>%
  mutate_at(vars(Sensitivity_Car, Sensitivity_Scooter), ~ifelse(is.na(.),Sensitivity_Motorized, .))%>%
  mutate_at(vars(Sensitivity_BikeElectric, Sensitivity_BikeNonElectric), ~ifelse(is.na(.),Sensitivity_Bikecombined, .))%>%
  mutate_at(vars(Sensitivity_Metro, Sensitivity_Bus, Sensitivity_Train, Sensitivity_Tram), ~ifelse(is.na(.),Sensitivity_Publiccombined, .)) %>%
  mutate(Sensitivity_BikeElectric =ifelse(as.character(models) %in% c("5", "6","7","8","9"),NA, Sensitivity_BikeElectric))%>%
  mutate(Sensitivity_BikeNonElectric =ifelse(as.character(models) %in% c( "5","6", "7","8","9"),NA, Sensitivity_BikeNonElectric))%>%
  mutate(Sensitivity_Car =ifelse(as.character(models) %in% c( "6","7","8","9"),NA, Sensitivity_Car))%>%
  mutate(Sensitivity_Scooter =ifelse(as.character(models) %in% c( "6","7","8","9"),NA, Sensitivity_Scooter)) %>%
  mutate(Sensitivity_Metro =ifelse(as.character(models)%in% c("7","8","9"),NA, Sensitivity_Metro))%>%
  mutate(Sensitivity_Bus =ifelse(as.character(models)%in% c("9"),NA, Sensitivity_Bus))%>%
  mutate(Sensitivity_Train =ifelse(as.character(models)%in% c("8", "9"),NA, Sensitivity_Train))%>%
  mutate(Sensitivity_Tram =ifelse(as.character(models)%in% c("7","8","9"),NA, Sensitivity_Tram))%>%
  gather(key = "key", value = "value", - models)%>%
  mutate(size2 = ifelse(key=="accuracy", "a" , ifelse(key=="macro_F1", "c", "b"))) %>%
  mutate(label_left  =  if_else(models==min(models), as.character(key), NA_character_), NA_character_)%>%
  mutate(label_right =  ifelse(key %in% c("Sensitivity_Motorized", 'Sensitivity_Bikecombined',"Sensitivity_Publiccombined"), 
                               if_else(models==max(models), as.character(key), NA_character_),NA_character_))



plot_models2 <- ggplot(models_summarized2) +
  geom_line(aes(x        = models, 
                y        = value, 
                col      = key,
                size     = size2,
                linetype = size2),
            alpha=.9)+
  geom_point(aes(x     = models, 
                 y     = value,
                 col   = key,
                 shape = key),
             size=2)+
  
  geom_label_repel(aes(label= label_left,
                       x= models, 
                       y =value),
                   nudge_x = -.5,
                   na.rm = TRUE)+
  geom_label_repel(aes(label= label_right,
                       x= models, 
                       y =value),
                   nudge_x = 1,
                   na.rm = TRUE)+
  scale_x_continuous(breaks= 1:9, 
                     labels=c("Only GPS &\n time features", 
                              "add context\nlocation features",
                              "add registry data", 
                              "collapse bike and e-bike",
                              "collapse Motorized modes",
                              "collapse tram and metro ",
                              "add train to collapsed\n public transport",
                              "add bus to collapsed\n public transport",
                              "exclude user error"),
                     limits=c(-1,11))+
  theme_minimal()+
  scale_shape_manual(values=2:16)+
  scale_size_manual(values=c("a"       = 2,
                             "b"       = .7,
                             "c"       = .7))+
  scale_linetype_manual(values=c("a"       = "solid",
                                 "c"       = "solid",
                                 "b"       = "dashed"))+
  theme(axis.text.x        = element_text(angle = 60, 
                                          hjust = 1),
        panel.grid.minor.x = element_blank())+
  scale_color_manual(values= c(accuracy = "black",
                               macro_F1 = "black",
                               Sensitivity_Car = "#FFD086",
                               Sensitivity_Metro ="#3E05EB",
                               Sensitivity_Tram="#1540BF",
                               Sensitivity_Bus ="#2877EB",
                               Sensitivity_BikeElectric ="#00FFBE",
                               Sensitivity_BikeNonElectric  ="#00CC36",
                               Sensitivity_Walk = "#8A1B00",
                               Sensitivity_Train="#87DFFF",
                               Sensitivity_Scooter = "#EBE800",
                               Sensitivity_Bikecombined = "#26803E",
                               Sensitivity_Publiccombined = "#740E80",
                               Sensitivity_Motorized  = "orange",
                               Sensitivity_User_error = "red"))

ggsave("plot_model9.svg", plot_models2 ,width= 30, height=18, units = "cm") # save as .svg to facilitate editting.
```

Feature importance
------------------

This is the code that was used to estimate feature importance. The Caret model, already calculates the variable (feature) importance, but this code is needed to estimate the significance using the Altman method.

``` r
ranger_p_value1 <- ranger::importance_pvalues(rf_output_vars1$finalModel, 
                                              method= "altmann", 
                                              formula= mode3~., 
                                           data=all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars1_sub5,
                                           num.permutations = 1000)


ranger_p_value123 <- ranger::importance_pvalues(rf_output_vars123$finalModel, 
                                              method= "altmann", 
                                              formula= mode3~., 
                                              data=all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5,
                                              num.permutations = 1000)

ranger_p_value9 <- ranger::importance_pvalues(rf_output_vars123_combinedpublic_noer$finalModel, 
                                              method= "altmann", 
                                              formula= mode3~., 
                                              data=all_trips_with_person_char_missing_fixed_with_dummies_combined_modesall_no_error_vars123_sub5,
                                              num.permutations = 1000)



varImp(rf_output_vars1)
varImp(rf_output_vars123)
varImp(rf_output_vars123_combinedpublic_noer)
```

### Marginal probability plots

With this code the marginal probabilities of three different features are calculated and plotted. We first create two functions that are needed to optimize and extract the marginal probabilities.

``` r
setDTthreads(7)

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5 <- readRDS("all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5.RDS")
rf_output_vars123<-readRDS("rf_output_vars123.RDS")

## these matching functions are needed.
CJ.table.1       <- function(X,Y) setkey(X[,c(k=1,.SD)],k)[Y[,c(k=1,.SD)],allow.cartesian=TRUE][,k:=NULL]
augment.multinom <- function(object, newdata) {
  newdata <- as_tibble(newdata)
  class_probs <- predict(object, newdata, type = "prob")
  bind_cols(newdata, as_tibble(class_probs))
}
## 1) ##

var <- quo(speed_0.9)
x_s <- select(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5, !!var)   # grid where we want partial dependencies
x_c <- select(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5, -!!var)  #  other predictors


setDT(x_s)
setDT(x_c)
grid<-CJ.table.1(x_s, x_c)

setDT(grid)
gridF1 <- grid[sample(.N, floor(18395517/20))]
au<-augment.multinom(rf_output_vars123, gridF1)

pd <- au %>%
  gather(class, prob, Train,BikeNonElectric,Car,Walk,Scooter,BikeElectric, PublicBus, User_error, Metro,Tram ) %>% 
  group_by(class, !!var) %>%
  summarize(marginal_prob = mean(prob))

saveRDS(pd, "pd_speed90_model3.rds")

## 2) ##

var <- quo(intrapolated_near_traintracks_relative1)

x_s <- select(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5, !!var)   # grid where we want partial dependencies
x_c <- select(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5, -!!var)  #  other predictors
grid<-crossing(x_s, x_c)
setDT(grid)
setDT(x_s)
setDT(x_c)
grid<-CJ.table.1(x_s, x_c)

setDT(grid)
gridF1 <- grid[sample(.N, floor(18395517/20))]

au<-augment.multinom(rf_output_vars123, gridF1)
gc()

pd <- au %>%
  gather(class, prob, Train,BikeNonElectric,Car,Walk,Scooter,BikeElectric, PublicBus, User_error, Metro,Tram ) %>% 
  group_by(class, !!var) %>%
  summarize(marginal_prob = mean(prob))


saveRDS(pd, "pd_traintrack.rds")

## 3) ##
var <- quo(near_subway_station_relative2)

x_s <- select(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5, !!var)   # grid where we want partial dependencies
x_c <- select(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5, -!!var)  #  other predictors
setDT(x_s)
setDT(x_c)
grid<-CJ.table.1(x_s, x_c)


setDT(grid)
gridF1 <- grid[sample(.N, floor(18395517/10))]
au<-augment.multinom(rf_output_vars123, gridF1)
gc()


pd <- au %>%
  gather(class, prob, Train,BikeNonElectric,Car,Walk,Scooter,BikeElectric, PublicBus, User_error, Metro,Tram ) %>% 
  group_by(class, !!var) %>%
  summarize(marginal_prob = mean(prob))

saveRDS(pd, "pd_subway.rds")
```

#### make plot

The above code calculated the marginal probabilities, This code plots them, like figure 5

``` r
pd_traintrack <-readRDS("pd_traintrack.rds")
pd_subway <-readRDS("pd_subway.rds")
pd_speed <-readRDS("pd_speed90_model3.rds")

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5 <- readRDS("all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5.RDS")


plotforlegend <- tibble(x= 1:200,
                        y= 2:201,
                        modes =sample(unique(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5$mode3),
                                      200, replace = TRUE)) %>%
  ggplot(aes( x ,    y, color = modes)) +
  geom_smooth(size = 1.5,  
              se = FALSE, 
              alpha=.9)+
  scale_color_manual(values= c(Car = "#FFD086",
                               Metro ="#3E05EB",
                               Tram="#1540BF",
                               PublicBus ="#2877EB",
                               BikeElectric ="#00FFBE",
                               BikeNonElectric  ="#00CC36",
                               Walk = "#8A1B00",
                               Train="#87DFFF",
                               Scooter = "#EBE800",
                               User_error = "red"))+ 
  theme_minimal()+  
  theme(legend.position = "bottom")+
  guides(col=guide_legend(nrow=2,byrow=TRUE))
legend <- cowplot::get_legend(plotforlegend)

plot_traintrack<- pd_traintrack %>%
  ggplot(aes(intrapolated_near_traintracks_relative1, marginal_prob, color = class)) +
  geom_smooth(size = 1.5,  
              se = FALSE, 
              alpha=.9) +
  labs(y = "Average class probability across all other predictors",
       x = "proximity train track") +
  coord_cartesian(xlim = c(0,.15))+
  theme_minimal()+
  scale_color_manual(values= c(Car = "#FFD086",
                               Metro ="#3E05EB",
                               Tram="#1540BF",
                               PublicBus ="#2877EB",
                               BikeElectric ="#00FFBE",
                               BikeNonElectric  ="#00CC36",
                               Walk = "#8A1B00",
                               Train="#87DFFF",
                               Scooter = "#EBE800",
                               User_error = "red"))+
  theme(legend.position = "none")

plot_subway<- pd_subway %>%
  ggplot(aes(near_subway_station_relative2, marginal_prob, color = class)) +
  geom_smooth(size = 1.5,  
              se = FALSE, 
              alpha=.9) +
  labs(y = "",
       x = "near_subway_station_relative2") +
  coord_cartesian(xlim = c(0,.2))+
  theme_minimal()+
  scale_color_manual(values= c(Car = "#FFD086",
                               Metro ="#3E05EB",
                               Tram="#1540BF",
                               PublicBus ="#2877EB",
                               BikeElectric ="#00FFBE",
                               BikeNonElectric  ="#00CC36",
                               Walk = "#8A1B00",
                               Train="#87DFFF",
                               Scooter = "#EBE800",
                               User_error = "red"))+
  theme(legend.position = "none")

plot_speed<- pd_speed %>%
  ggplot(aes(speed_0.9, marginal_prob, color = class)) +
  geom_smooth(size = 1.5,  
              se = FALSE, 
              alpha=.9) +
  labs(y = "",
       x = "speed_0.9") +
  #coord_cartesian(xlim = c(0,quantile(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5$speed_0.9, .98)))+
  xlim(0, 40)+
  theme_minimal()+
  scale_color_manual(values= c(Car = "#FFD086",
                               Metro ="#3E05EB",
                               Tram="#1540BF",
                               PublicBus ="#2877EB",
                               BikeElectric ="#00FFBE",
                               BikeNonElectric  ="#00CC36",
                               Walk = "#8A1B00",
                               Train="#87DFFF",
                               Scooter = "#EBE800",
                               User_error = "red"))+
  theme(legend.position = "none")


toprow <- cowplot::plot_grid(plot_traintrack,plot_subway , plot_speed, labels = c("A",'B', 'C'), align = 'v',
                             ncol = 3)


plot_marginal_probs2 <- cowplot::plot_grid(toprow, legend,
                                          ncol = 1,
                                          rel_heights = c(10, 1))


ggsave("plot_marginal_probs2.svg", plot_marginal_probs2, width= 25, units = "cm")
```

code to calculate summary statics.
----------------------------------

This is the code to calculate the statistics presented in table 7 and 8 of the thesis.

``` r
analyzed_files <- list.files("./output_files/analyzed_unlabelled_tracks")
analyzed_files_intrapolated <- list.files("./output_files/analyzed_unlabelled_tracks_int")


analyzed_files_list <- list()
for(i in analyzed_files[analyzed_files%in%analyzed_files_intrapolated]) {
  analyzed_files_list[[i]] <-
    readRDS(paste0("./output_files/analyzed_unlabelled_tracks/", i))
}

analyzed_files_df <- rbindlist(analyzed_files_list, fill = TRUE)


analyzed_files_turns <- list.files("./output_files/analyzed_unlabelled_tracks_turns")


analyzed_files_list_turn <- list()
for(i in analyzed_files_turns[analyzed_files_turns%in%analyzed_files_intrapolated]) {
  analyzed_files_list_turn[[i]] <-
    readRDS(paste0("./output_files/analyzed_unlabelled_tracks_turns/", i))
}

analyzed_files_turns_df <- rbindlist(analyzed_files_list_turn, fill = TRUE)
analyzed_files_df <-left_join(analyzed_files_df, select(analyzed_files_turns_df, -n_segments, -mode3, -n_datapoints, -ratio_filtered, - device_id, - active_modes ))



setwd("//cbsp.nl/productie/Projecten/BPM/301707WaarnVnw_SEC1/Werk/Tabi Verplaatsingen App/Data/Laurent/copy GPS data Laurent")
#all_trips <- readRDS("./output_files/all_trips.RDS")
aggdata <- read_sav("./external_imput_data/user_info/AVA-EVA Analysis data set - complete.sav")
setDT(aggdata)

# Changing variable names
setnames(aggdata, "LFT", "Age_in_yrs")
setnames(aggdata, "AUTOP", "Has_car")
setnames(aggdata, "BROMMEROP", "Has_moped")
setnames(aggdata, "RIJBEWIJSP", "Driverslicense")
setnames(aggdata, "STEDBUURT", "Urbanity")
setnames(aggdata, "INHP100HGEST", "Income")
setnames(aggdata, "INHP100HBEST", "Income2")

aggdata2 <- aggdata[, c("username",
                        "Geslacht" ,  
                        "Age_in_yrs",  
                        "Has_car", 
                        "Urbanity", 
                        "Income", 
                        "Income2", 
                        "Has_moped" ,
                        "Driverslicense", 
                        "LEASEAUTO",
                        "STEDGEM",
                        "INPSECJ" )]


aggdata2 <- aggdata2 %>%
  as_tibble() %>%
  mutate(username = as.character(username))

# 1) list all the folders in the Imput_data folder
folders<-list.dirs(path ="./Imput_data",
                   full.names=F,
                   recursive = F)

# 2) create an empty list to store the transport modes in
users_list <- list()

for(i in 1:length(folders)){
  users_list[[i]]<-fread(paste0(getwd(), "/Imput_data/",
                                folders[i], 
                                "/users.csv"))
}


users_1101_0220 <-rbindlist(users_list) %>%
  select(id, username) %>%
  distinct()%>%
  rename("user_id" = id)


device_list <- list()

for(i in 1:length(folders)){
  device_list[[i]]<-fread(paste0(getwd(), "/Imput_data/",
                                 folders[i], 
                                 "/devices.csv"))
}


device__1101_0220 <-rbindlist(device_list) %>%
  select(-created_at, -updated_at, -deleted_at) %>%
  distinct()%>%
  rename("device_id" = id)

joined_user_device_info <- left_join(device__1101_0220, users_1101_0220, by = "user_id")
setDT(joined_user_device_info)


all_user_info <- left_join(joined_user_device_info, aggdata2, by = "username")

setDT(all_user_info)
setDT(analyzed_files_df)


all_trips_with_person_char <- left_join(analyzed_files_df, all_user_info, by = "device_id")
saveRDS(all_trips_with_person_char, "output_files/all_trips_with_person_char_unlabelled.RDS")

#rm(list=ls())
#saveRDS(all_trips_with_person_char, "output_files/all_trips_with_person_char.RDS")
#########
all_trips_with_person_char<-readRDS("output_files/all_trips_with_person_char_unlabelled.RDS")

analyzed_files_intrapolated <- list.files("./output_files/analyzed_unlabelled_tracks_int")


analyzed_files_list_intrapolated <- list()
for(i in analyzed_files_intrapolated) {
  analyzed_files_list_intrapolated[[i]] <-
    readRDS(paste0("./output_files/analyzed_unlabelled_tracks_int/", i))
}

analyzed_files_df_intrapolated <- rbindlist(analyzed_files_list_intrapolated, fill = TRUE)

analyzed_files_intrapolated_turn <- list.files("./output_files/analyzed_unlabelled_tracks_int_turn")

#analyzed_files_intrapolated_turn[1:3]



analyzed_files_list_intrapolated_turn <- list()
for(i in analyzed_files_intrapolated_turn) {
  analyzed_files_list_intrapolated_turn[[i]] <-
    readRDS(paste0("./output_files/analyzed_unlabelled_tracks_int_turn/", i))
}

analyzed_files_df_intrapolated_turn <- rbindlist(analyzed_files_list_intrapolated_turn, fill = TRUE)

analyzed_files_df_intrapolated_turn$TrackDataIdVar <- as.numeric(gsub( '.{8}$', '',substr(analyzed_files_intrapolated_turn, 29, 100)))
analyzed_files_df_intrapolated$TrackDataIdVar <- as.numeric(gsub( '.{8}$', '',substr(analyzed_files_intrapolated, 29, 100)))


analyzed_files_df_intrapolated <-left_join(analyzed_files_df_intrapolated, select(analyzed_files_df_intrapolated_turn, -n_segments, - active_modes,-n_turn_5plus150))

# 
# analyzed_files_df_intrapolated <- analyzed_files_df_intrapolated %>%
#   select(-mean_dist_betweenpoint,
#          -mean_turn,
#          -median_turn,
#          -n_turn_1plus90,
#          -n_turn_1plus150,
#          -n_turn_5plus90,
#          -n_turn_5plus150,
#          -turn_1plus90_ps,
#          -turn_1plus150_ps,
#          -turn_5plus90_ps,
#          -turn_1plus150_ps)
# 

for(i in 1:ncol(analyzed_files_df_intrapolated)){
  names(analyzed_files_df_intrapolated)[i] <- paste0("intrapolated_",names(analyzed_files_df_intrapolated)[i])
}


vars_to_delete <- c("os_version", "username",  "user_id", "manufacturer", "model", "os", "device_id" , "active_modes", "TrackDataIdVar")
vars_to_delete2 <- c("os_version",  "user_id", "manufacturer", "model", "os", "device_id" , "active_modes", "TrackDataIdVar")

vars_level1    <- c("mid_point", "n_segments", "n_datapoints",  "distance_straight_line","total_distance.segments" ,
                    "total_distance", "average_Speed_track","mean_bearing","median_bearing" ,                             
                    "max_bearing", "bearing_0.05" , "bearing_0.25" ,                               
                    "bearing_0.75", "bearing_0.9", "bearing_0.95",                               
                    "bearing_0.99" ,"bearing_sd"  ,"mean_abs.bearing",                            
                    "median_abs.bearing","max_abs.bearing","abs.bearing_0.05",                           
                    "abs.bearing_0.25","abs.bearing_0.75" ,"abs.bearing_0.9",                          
                    "abs.bearing_0.95" ,"abs.bearing_0.99","abs.bearing_sd",                              
                    "mean_rel_bearing" ,  "median_rel_bearing","max_rel_bearing" ,                           
                    "rel_bearing_0.05" ,"rel_bearing_0.25","rel_bearing_0.75",                          
                    "rel_bearing0.9","rel_bearing_0.95","rel_bearing_0.99",                            
                    "rel_bearing_sd", "mean_speed","median_speed",                                
                    "max_speed","speed_0.05","speed_0.25",                                  
                    "speed_0.75" ,"speed_0.9","speed_0.95",                                  
                    "speed_0.99", "speed_sd" ,"speed_skew",   
                    "speed_above80_ratio" ,"speed_above120_ratio", "speed_below5_ratio",                          
                    "n_inf_acc","mean_acceleration","median_acceleration",                         
                    "max_acceleration","min_acceleration","acceleration_0.01" ,                          
                    "acceleration_0.05","acceleration_0.1","acceleration_0.25",                           
                    "acceleration_0.75","acceleration_0.9","acceleration_0.95",                           
                    "acceleration_0.99","acceleration_sd","acceleration_skew" ,                          
                    "mean_dist_betweenpoint","mean_turn","median_turn",                                 
                    "n_turn_1plus90","n_turn_1plus150","n_turn_5plus90" ,                             
                    "n_turn_5plus150","turn_1plus90_ps","turn_1plus150_ps",                            
                    "turn_5plus90_ps","turn_5plus150_ps", "ratio_straightline_distance.total_distance","diff_time_total")
vars_level2    <- c("n_urban_area","unique_urban_area", "ratio_urban_area" , "mean_bev_dens" ,"median_bev_dens" ,"mean_urban" , "sdBEV", "median_urban" ,  "mode_urban" ,
                    "ratio_in_urban","gpsPoints_nearby_train_station"  , "mean_distance_train_station","median_distance_train_station",  "gpsPoints_nearby_train_station_start",
                    "gpsPoints_nearby_train_station_end" ,"gpsPoints_nearby_station_start_and_end","unique_train_stations", "near_trainstation_relative1","near_trainstation_relative2",
                    "gpsPoints_nearby_train_track" ,"mean_distance_traintracks", "median_distance_traintracks" ,"near_traintracks_relative1","near_traintracks_relative2", 
                    "gpsPoints_nearby_highways" ,"mean_distance_highways","median_distance_highways" , "near_highways_relative1","near_highways_relative2", "gpsPoints_nearby_onofframps" ,
                    "mean_distance_onofframps", "median_distance_onofframps" ,"unique_train_onofframps", "near_onofframps_relative1" ,"near_onofframps_relative2", 
                    "gpsPoints_nearby_subway_station","mean_distance_subway_station" , "median_distance_subway_station","gpsPoints_nearby_subway_station_start",
                    "gpsPoints_nearby_subway_station_end", "gpsPoints_nearby_subwaystation_start_and_end", "unique_subway_stations","near_subway_station_relative1" , 
                    "near_subway_station_relative2","gpsPoints_nearby_tram_stop", "mean_distance_tram_stop","median_distance_tram_stop" ,"gpsPoints_nearby_tram_stop_start",
                    "gpsPoints_nearby_tram_stop_end", "gpsPoints_nearby_tram_stop_start_and_end" ,"unique_tram_stop"  , "near_tram_stop_relative1" ,"near_tram_stop_relative2" ,
                    "gpsPoints_nearby_tram_track", "mean_distance_tram_track" , "median_distance_tram_track", "near_tramtracks_relative1","near_tramtracks_relative2" , 
                    "gpsPoints_nearby_subway_track" , "mean_distance_subway_track","median_distance_subway_track" , "near_subwaytrack_relative1","near_subwaytrack_relative2",
                    "gpsPoints_nearby_bus_stops", "mean_distance_bus_stops","median_distance_bus_stops", "gpsPoints_nearby_bus_stops_start",            
                    "gpsPoints_nearby_bus_stops_end" , "gpsPoints_nearby_bus_stops_start_and_end", "unique_bus_stops" ,                           
                    "near_bus_stops_relative1" ,"near_bus_stops_relative2","during_rush", "mean_accuracy" ,"ratio_filtered" )
vars_level3    <- c( "Geslacht","Age_in_yrs","Has_car", "Urbanity", "Income","Income2", "Has_moped","Driverslicense" ,"LEASEAUTO","INPSECJ", "STEDGEM" )



vars_to_delete_dummy <- c(paste0("dummyNA_",vars_to_delete),"dummyNA_mode3")
vars_level1_dummy <- paste0("dummyNA_",vars_level1)
vars_level2_dummy <- paste0("dummyNA_",vars_level2)
vars_level3_dummy <- paste0("dummyNA_",vars_level3)
vars_to_delete_dummy <- vars_to_delete_dummy[vars_to_delete_dummy!="dummyNA_TrackDataIdVar"]
vars_to_delete_intrapolated <- c(paste0("intrapolated_",vars_to_delete), "intrapolated_active_modes", "intrapolated_mode3", "intrapolated_TrackDataIdVar")
vars_level1_intrapolated <- paste0("intrapolated_",vars_level1)
vars_level2_intrapolated <- paste0("intrapolated_",vars_level2)
vars_level3_intrapolated <- paste0("intrapolated_",vars_level3)


matrix_dummies <- as.data.frame(matrix(NA, ncol= ncol(all_trips_with_person_char), nrow= nrow(all_trips_with_person_char)))

#for(i in c(1, 3, 5:ncol(all_trips_with_person_char))){
for(i in 1:ncol(all_trips_with_person_char)){
  var<-all_trips_with_person_char[,i]
  matrix_dummies[,i] <- is.na(var)
  names(matrix_dummies)[i] <- paste0("dummyNA_",names(all_trips_with_person_char)[i])
}
# matrix_dummies[,2] <- all_trips_with_person_char[,2]
# matrix_dummies[,4] <- all_trips_with_person_char[,4]
# names(matrix_dummies)[2]<- paste0("dummyNA_",names(all_trips_with_person_char)[2])
# names(matrix_dummies)[4]<- paste0("dummyNA_",names(all_trips_with_person_char)[4])


all_trips_with_person_char_missing_fixed <- all_trips_with_person_char %>%
  mutate(ratio_straightline_distance.total_distance = ifelse(is.na(ratio_straightline_distance.total_distance),1, ratio_straightline_distance.total_distance))%>%
  mutate(average_Speed_track = ifelse(is.na(average_Speed_track), 0, average_Speed_track))%>%
  mutate(median_speed = ifelse(is.na(median_speed), 0, median_speed))%>%
  mutate(mean_speed = ifelse(is.na(mean_speed), 0, mean_speed))%>%
  mutate_if(grepl("_acceleration", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("bearing", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("_relative1", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("max_speed", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("acceleration_", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_acceleration", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("bearing", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("speed_", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_speed", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_sd", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("mean_distance_", names(.)), ~if_else(is.na(.), 150, .))%>%
  mutate_if(grepl("median_distance_", names(.)), ~if_else(is.na(.), 150, .)) %>%
  mutate_if(grepl("_relative1", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate_if(grepl("_ps", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate_if(grepl("mean_dist_betweenpoint", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate(mean_bev_dens = ifelse(is.na(mean_bev_dens), median(all_trips_with_person_char$mean_bev_dens, na.rm=TRUE), mean_bev_dens))%>%
  mutate(median_bev_dens = ifelse(is.na(median_bev_dens), median(all_trips_with_person_char$median_bev_dens, na.rm=TRUE), median_bev_dens))%>%
  mutate(sdBEV = ifelse(is.na(sdBEV), median(all_trips_with_person_char$sdBEV, na.rm=TRUE), sdBEV))%>%
  mutate(median_urban = ifelse(is.na(median_urban), median(all_trips_with_person_char$median_urban, na.rm=TRUE), median_urban))%>%
  mutate(mode_urban = ifelse(is.na(mode_urban), median(all_trips_with_person_char$mode_urban, na.rm=TRUE), mode_urban))%>%
  mutate(ratio_in_urban = ifelse(is.na(ratio_in_urban), median(all_trips_with_person_char$ratio_in_urban, na.rm=TRUE), ratio_in_urban))%>%
  mutate(mean_urban = ifelse(is.na(mean_urban), median(all_trips_with_person_char$mean_urban, na.rm=TRUE), mean_urban))%>%
  mutate(Age_in_yrs = as.numeric(Age_in_yrs))%>%
  mutate(Has_car = as.numeric(Has_car))%>%
  mutate(Urbanity = as.numeric(Urbanity))%>%
  mutate(Income2 = as.numeric(Income2))%>%
  mutate(Income = as.numeric(Income))%>%
  mutate(LEASEAUTO = as.numeric(LEASEAUTO))%>%
  mutate(Driverslicense = as.numeric(Driverslicense))%>%
  mutate(STEDGEM = as.numeric(STEDGEM))%>%
  mutate(INPSECJ = as.numeric(INPSECJ))%>%
  mutate(Has_moped = as.numeric(Has_moped))

analyzed_files_df_intrapolated_missing_fixed <- analyzed_files_df_intrapolated %>%
  mutate(intrapolated_ratio_straightline_distance.total_distance = ifelse(is.na(intrapolated_ratio_straightline_distance.total_distance),1, intrapolated_ratio_straightline_distance.total_distance))%>%
  mutate(intrapolated_average_Speed_track = ifelse(is.na(intrapolated_average_Speed_track), 0, intrapolated_average_Speed_track))%>%
  mutate(intrapolated_median_speed = ifelse(is.na(intrapolated_median_speed), 0, intrapolated_median_speed))%>%
  mutate(intrapolated_mean_speed = ifelse(is.na(intrapolated_mean_speed), 0, intrapolated_mean_speed))%>%
  mutate_if(grepl("_acceleration", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("bearing", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("_relative1", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("max_speed", names(.)), ~if_else(!is.finite(.), 0, .))%>%
  mutate_if(grepl("acceleration_", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_acceleration", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("bearing", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("speed_", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_speed", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("_sd", names(.)), ~if_else(is.na(.), 0, .))%>%
  mutate_if(grepl("mean_distance_", names(.)), ~if_else(is.na(.), 150, .))%>%
  mutate_if(grepl("median_distance_", names(.)), ~if_else(is.na(.), 150, .)) %>%
  mutate_if(grepl("_relative1", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate_if(grepl("_ps", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate_if(grepl("mean_dist_betweenpoint", names(.)), ~if_else(is.na(.), 0, .)) %>%
  mutate(intrapolated_mean_bev_dens = ifelse(is.na(intrapolated_mean_bev_dens), median(analyzed_files_df_intrapolated$intrapolated_mean_bev_dens, na.rm=TRUE), intrapolated_mean_bev_dens))%>%
  mutate(intrapolated_median_bev_dens = ifelse(is.na(intrapolated_median_bev_dens), median(analyzed_files_df_intrapolated$intrapolated_median_bev_dens, na.rm=TRUE), intrapolated_median_bev_dens))%>%
  mutate(intrapolated_sdBEV = ifelse(is.na(intrapolated_sdBEV), median(intrapolated_analyzed_files_df_intrapolated$intrapolated_sdBEV, na.rm=TRUE), intrapolated_sdBEV))%>%
  mutate(intrapolated_median_urban = ifelse(is.na(intrapolated_median_urban), median(analyzed_files_df_intrapolated$intrapolated_median_urban, na.rm=TRUE), intrapolated_median_urban))%>%
  mutate(intrapolated_mode_urban = ifelse(is.na(intrapolated_mode_urban), median(analyzed_files_df_intrapolated$intrapolated_mode_urban, na.rm=TRUE), intrapolated_mode_urban))%>%
  mutate(intrapolated_ratio_in_urban = ifelse(is.na(intrapolated_ratio_in_urban), median(analyzed_files_df_intrapolated$intrapolated_ratio_in_urban, na.rm=TRUE), intrapolated_ratio_in_urban))%>%
  mutate(intrapolated_mean_urban = ifelse(is.na(intrapolated_mean_urban), median(analyzed_files_df_intrapolated$intrapolated_mean_urban, na.rm=TRUE), intrapolated_mean_urban))


all_trips_with_person_char_missing_fixed_with_dummies <- bind_cols(all_trips_with_person_char_missing_fixed, matrix_dummies)

#sum(all_trips_with_person_char_missing_fixed_with_dummies$TrackDataIdVar != all_trips_with_person_char_missing_fixed_with_dummies$dummyNA_TrackDataIdVar)


all_trips_with_person_char_missing_fixed_with_dummies_and_intrap <- bind_cols(all_trips_with_person_char_missing_fixed_with_dummies,
                                                                              analyzed_files_df_intrapolated_missing_fixed)

sum(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap$device_id != all_trips_with_person_char_missing_fixed_with_dummies_and_intrap$intrapolated_device_id) 

sum(na.omit(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap$TrackDataIdVar != all_trips_with_person_char_missing_fixed_with_dummies_and_intrap$intrapolated_TrackDataIdVar)) 

test_set <- readRDS("test_set.RDS")

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap2 <- left_join(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap, test_set)


all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap2 %>%
  mutate(total_trips = BikeElectric_used+BikeNonElectric_used+Car_used+Lorry_used+Metro_used+Motorbike_used+ multi_used+ other_transport_mode_used+ PublicBus_used+Train_used+
           `trip saved twice with multi modes_used`+ `User error_used`+ Walk_used)



#check this again, once complete!!
sum(!is.na(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3$Tram_used))/nrow(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3)


all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3[,500:514] <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3[,500:514]/all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3[,515]

all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3<-all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3[,1:514] 

unlabbeled_stats <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3



saveRDS(unlabbeled_stats, "unlabbeled_stats.RDS")
#rm(list=ls())


load("//cbsp.nl/productie/Projecten/BPM/301707WaarnVnw_SEC1/Werk/Tabi Verplaatsingen App/Data/Laurent/copy GPS data Laurent/model 3 and 9 with all trips used3.RData")


unlabbeled_stats<- readRDS("unlabbeled_stats.RDS")

unlabbeled_stats_vars123 <- unlabbeled_stats %>%
  select(-vars_to_delete, 
         -vars_to_delete_dummy,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar,
         -mode3)  # this is needed, because no spaces are allowed in the algorithm 




unlabbeled_stats_vars123_PP <- preProcess(unlabbeled_stats_vars123, method= "medianImpute")
 # unlabbeled_stats_vars123_PP <- preProcess(unlabbeled_stats_vars123, method= "bagImpute") # yields same results

unlabbeled_stats_vars123 <- predict(unlabbeled_stats_vars123_PP, 
                                    unlabbeled_stats_vars123)


unlabbeled_stats_vars123_sub5 <- unlabbeled_stats_vars123 %>%
  filter(n_datapoints > 2)#%>%
#   dplyr::rename(trip_saved_twice_with_multi_modes_used =`trip saved twice with multi modes_used`  ,
#           User_error_used = `User error_used`)
names(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5)[!(names(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5) %in% names(unlabbeled_stats_vars123_sub5))]



predict(rf_output_vars123.ratio, unlabbeled_stats_vars123_sub5, type= "prob")

predicted_values <- enframe(predict(rf_output_vars123.ratio, unlabbeled_stats_vars123_sub5))

predicted_all_unlabelled<-predicted_values %>%
  group_by(value) %>%
  summarise(tt =n()/nrow(predicted_values)*100)

predicted_all_unlabelled <- predicted_all_unlabelled %>%
  mutate(tt2 = ifelse(value !=  "User_error", tt*(100/(100-filter(predicted_all_unlabelled, value =="User_error")$tt)), tt))



recorded_all_labelled <-all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5%>%  
  group_by(mode3) %>%
  summarise(tt =n()/nrow(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5)*100)

recorded_all_labelled <- recorded_all_labelled %>%
  mutate(tt2 = ifelse(mode3 !=  "User_error", tt*(100/(100-filter(recorded_all_labelled, mode3 =="User_error")$tt)), tt))



labelled_unlabelled_combined <- c(as.character(predicted_values$value), all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5$mode3) %>% 
  enframe()%>%
  group_by(value) %>%
  summarise(tt =n()/(nrow(predicted_values)+nrow(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5))*100)


labelled_unlabelled_combined <- labelled_unlabelled_combined %>%
    mutate(tt2 = ifelse(value !=  "User_error", tt*(100/(100-filter(labelled_unlabelled_combined, value =="User_error")$tt)), tt))
  


####
data3 <- read.spss("./external_imput_data/other/ODiN 2018 responsdata.sav")
setDT(data3)
data4 <- data3[, c("Verpl_Vertrektijd", "Verpl_Aankomsttijd", "Rit_Vervoerwijze", "Rit_Afstand", "Rit_Afstandseenheid",  "Rit_AflConv_AflConVervoerwijze", "Veilignummer" )]


all_ODIN_ODIN  <- data4 %>%
  as_tibble()%>%
  mutate(Rit_AflConv_AflConVervoerwijze = as.character(Rit_AflConv_AflConVervoerwijze)) %>%
  filter(Rit_AflConv_AflConVervoerwijze %in% c("Personenauto",
                                               "Trein", 
                                               "Bus", 
                                               "Tram", 
                                               "metro", 
                                               "Speed-pedelec", 
                                               "Elektrische Fiets", 
                                               "Niet-elektrische fiets",
                                               "Te voet",
                                               "Bromfiets",
                                               "Snorfiets"))%>%
  mutate(Rit_AflConv_AflConVervoerwijze = ifelse(Rit_AflConv_AflConVervoerwijze %in% c("Elektrische Fiets", "Speed-pedelec"), "E-bike", Rit_AflConv_AflConVervoerwijze)) %>%
  mutate(Rit_AflConv_AflConVervoerwijze = ifelse(Rit_AflConv_AflConVervoerwijze %in% c("Bromfiets", "Snorfiets"), "scooter", Rit_AflConv_AflConVervoerwijze)) %>%
  mutate(distance = ifelse(Rit_Afstandseenheid== "Kilometer", Rit_Afstand*1000, Rit_Afstand)) %>%
  mutate(total_distanceunderx = ifelse(distance<501, 1,0))%>%
  mutate(diff_time_total = as_datetime(paste0(substr(Verpl_Aankomsttijd,1,8), " 1970-01-01"), format="%H:%M:%S %Y-%d-%m")- 
           as_datetime(paste0(substr(Verpl_Vertrektijd,1,8), " 1970-01-01"), format="%H:%M:%S %Y-%d-%m")) %>%  
  mutate(diff_time_total= ifelse(diff_time_total<0, diff_time_total+ 86400, diff_time_total)) %>%
  group_by(Rit_AflConv_AflConVervoerwijze)%>%
  summarise(ratio= n()/ 165635*100)


####

veilinummerODIN <- data4 %>%
  as_tibble() %>%
  mutate(Veilignummer =gsub(" ", "", Veilignummer))


VN_and_username <- aggdata%>%
  select(Veilignummer, username) %>%
  mutate(username = as.character(username))%>%
  filter(Veilignummer %in% veilinummerODIN$Veilignummer)

labelled_GPS_ODIN <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3 %>%
  filter(username %in% VN_and_username$username)%>%  
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3)) %>%
  filter(n_datapoints > 2) %>%
  group_by(mode3) %>%
  summarise(tt =n()/2584*100)

labelled_GPS_ODIN <- labelled_GPS_ODIN %>%
  mutate(tt2 = ifelse(mode3 !=  "User_error", tt*(100/(100-filter(labelled_GPS_ODIN, mode3 =="User_error")$tt)), tt))



vars_to_delete2 <- vars_to_delete[vars_to_delete!="username"]

unlabbeled_stats_vars123.withusername <- unlabbeled_stats %>%
  select(-vars_to_delete2, 
         -vars_to_delete_dummy,
         -intrapolated_device_id,
         -intrapolated_active_modes,
         -intrapolated_mode3, 
         -intrapolated_TrackDataIdVar,
         -mode3)  # this is needed, because no spaces are allowed in the algorithm 

unlabbeled_stats_vars123.withusername_PP <- preProcess(unlabbeled_stats_vars123.withusername, method= "medianImpute")

unlabbeled_stats_vars123.withusername <- predict(unlabbeled_stats_vars123.withusername_PP, 
                                                 unlabbeled_stats_vars123.withusername)

anyNA(unlabbeled_stats_vars123.withusername)

unlabbeled_stats_vars123.withusername_sub5.ODIN <- unlabbeled_stats_vars123.withusername %>%
  filter(n_datapoints > 2)%>%
  filter(username %in% VN_and_username$username)

predicted_values_ODIN <- enframe(predict(rf_output_vars123.ratio, unlabbeled_stats_vars123.withusername_sub5.ODIN))

predicted_ODIN_unlabelled<-predicted_values_ODIN %>%
  group_by(value) %>%
  summarise(tt =n()/nrow(predicted_values_ODIN)*100)

predicted_ODIN_unlabelled <- predicted_ODIN_unlabelled %>%
  mutate(tt2 = ifelse(value !=  "User_error", tt*(100/(100-filter(predicted_ODIN_unlabelled, value =="User_error")$tt)), tt))


labelled_ODIN_modes <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap3 %>%
  filter(username %in% VN_and_username$username)%>%  
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  mutate(mode3 = ifelse(mode3== "User error", "User_error", mode3)) %>%
  filter(n_datapoints > 2) 


labelled_unlabelled_combined_ODIN_respondents <- c(as.character(predicted_values_ODIN$value), labelled_ODIN_modes$mode3) %>% 
  enframe()%>%
  group_by(value) %>%
  summarise(tt =n()/(nrow(predicted_values_ODIN)+nrow(labelled_ODIN_modes))*100)



labelled_unlabelled_combined_ODIN_respondents <- labelled_unlabelled_combined_ODIN_respondents %>%
  mutate(tt2 = ifelse(value !=  "User_error", tt*(100/(100-filter(labelled_unlabelled_combined_ODIN_respondents, value =="User_error")$tt)), tt))

######
######
all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5%>%
  mutate(total_distanceunderx = ifelse(total_distance<500, 1,0))%>%
  filter(mode3 != 'User_error') %>%
  summarise(median_dist = median(total_distance),
            mean_datapoint = median(n_datapoints),
            time= median(diff_time_total),
            shorttrip = mean(total_distanceunderx))



unlabbeled_stats_vars123_sub5%>%
  mutate(total_distanceunderx = ifelse(total_distance<500, 1,0))%>%
  filter(mode3 != 'User_error') %>%
  summarise(median_dist = median(total_distance),
            mean_datapoint = median(n_datapoints),
            time= median(diff_time_total),
            shorttrip = mean(total_distanceunderx))

bind_rows(unlabbeled_stats_vars123_sub5, all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5)%>%
  mutate(total_distanceunderx = ifelse(total_distance<500, 1,0))%>%
  filter(mode3 != 'User_error') %>%
  summarise(median_dist = median(total_distance),
            mean_datapoint = median(n_datapoints),
            time= median(diff_time_total),
            shorttrip = mean(total_distanceunderx))

bind_rows(unlabbeled_stats_vars123.withusername_sub5.ODIN, labelled_ODIN_modes)%>%
  mutate(total_distanceunderx = ifelse(total_distance<500, 1,0))%>%
  filter(mode3 != 'User_error') %>%
  summarise(median_dist = median(total_distance),
            mean_datapoint = median(n_datapoints),
            time= median(diff_time_total),
            shorttrip = mean(total_distanceunderx))
```

Logistic regression
-------------------

This is the code for the logistic regression presented in the thesis for the success of classification

``` r
all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5_2 <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123 %>%
  filter(n_datapoints > 2)

original <- all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123_sub5_2

original<- original %>%
  mutate(rowIndex = 1:n())

setDT(original)

pred3 <- predict(rf_output_vars123, original)


preds_and_original <- cbind(original, as.character(pred3))%>%
  mutate(classified = factor(ifelse(pred3==mode3, "correct", "incorrect" ), levels= c("incorrect", "correct"))) 


names(preds_and_original)

fullmodel3 <- glm(relevel(classified, ref= "incorrect") ~ n_datapoints + mean_accuracy + ratio_urban_area +
                    relevel(factor(mode3), ref= "Walk"),
                  data =preds_and_original, family =  binomial(link = "logit"))


stargazer(fullmodel3)
```

Code summary statistics
-----------------------

This code reproduces Table 9 from the thesis. First the top 10 most import features are selected, then they are grouped by transport mode, then statistics are calculated and then they are all out in one table.

``` r
all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123
setDT(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123)
names(all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123)

long_format_statistics <-  all_trips_with_person_char_missing_fixed_with_dummies_and_intrap_vars123 %>% 
  filter(mode3 != "Lorry"&
           mode3 != "trip saved twice with multi modes" &
           mode3 != "Motorbike" &
           mode3 != "other_transport_mode"&
           mode3 != "multi") %>%
  select("mode3",
         "intrapolated_near_traintracks_relative1",
         "near_subway_station_relative2",
         "speed_0.9",
         "near_subwaytrack_relative2",
         "speed_0.95",
         "intrapolated_near_tram_stop_relative1",
         "distance_straight_line",
         "speed_0.05",
         "speed_0.75",
         "median_speed") %>%
  gather(key= "statistic", value= "value", -mode3)


stats_of_Stat <- long_format_statistics %>%
  group_by(statistic, mode3) %>%
  summarise(mean   = mean(value, na.rm = TRUE),
            median = median(value, na.rm = TRUE),
            sd = sd(value, na.rm = TRUE),
            Q.75= quantile(value, .75, na.rm = TRUE),
            Q.25= quantile(value, .25, na.rm = TRUE))


data_table <- stats_of_Stat

seleced_stats <-  c(  "intrapolated_near_traintracks_relative1",
  "near_subway_station_relative2",
  "speed_0.9",
  "near_subwaytrack_relative2",
  "speed_0.95",
  "intrapolated_near_tram_stop_relative1",
  "distance_straight_line",
  "speed_0.05",
  "speed_0.75",
  "median_speed")

# ad a little rounding features that rounds to different decimals, based on the size of the statistic
myrounding <- function(x){
  if(x < 10){
    y <- round(x, 2)
  }else{
    if(x>10 & x<100){
      y <- round(x, 1)
    }else{
      y <- round(x, 0)}}
  
  return(y)
}

myroundingvec <- Vectorize(myrounding)


#Alternatvely, you could use the mean instead of the median for some of the statistics, by using this code:

# mean_instead_of_median <- c("intrapolated_near_traintracks_relative1",
#                             "near_subway_station_relative2",
#                             "near_subwaytrack_relative2",
#                             "intrapolated_near_tram_stop_relative1")


formatted_stats <- data_table %>%
  as_tibble() %>%
  mutate(CI_50_median = ifelse(statistic %in% mean_instead_of_median,
                               paste0(myroundingvec(mean), "\\newline [", myroundingvec(Q.25), "; ", myroundingvec(Q.75), "]"),
                               paste0(myroundingvec(median), "\\newline [", myroundingvec(Q.25), "; ", myroundingvec(Q.75), "]"))) %>%  select(statistic, mode3, CI_50_median) %>%
  spread(mode3, CI_50_median)%>%
  filter(statistic %in% seleced_stats)

xtable(formatted_stats)
```
