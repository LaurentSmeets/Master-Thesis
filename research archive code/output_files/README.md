After running the code this folder will atleast contain the following file;
- **all_trips.RDS**          (This are the feature level statistics for the labelled tracks. serves as imput to the Random Forest)
- **all_gpsdata.RDS** (this is simply all GPS data saved as an RDS)
- **all_unique_tracks_1101_0220.RDS** (This are all the trips that are filtered and corrected)
- **all_unique_tracks_1101_0220_with_transportmode.RDS**  (This are all the trips that are filtered and corrected of those trip that could be labelled to a transport mode)
- **locationwithtransport_full.RDS** (This is the location data of all trips, merged with the trip data, for all trips with a labelled mode)
- **completed_atleast_one.RDS** (these are all the device_ids that labelled at least one trip)
