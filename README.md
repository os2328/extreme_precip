# extreme_precip
Extreme Precipitation Catalog based on AORC

General idea: given a dataset of daily precipitation data (AORC, 1km spatial resolution), create a catalog of extreme precipitation events. To define an event, I plan to use precipitation intensity and spatial extent. The catalog should contain the hypercube of each event (date, lat, lon) and the event magnitude for the AORC data (to be able to identify top N events). The hypercube mask should be easily applicable to other datasets, considering potential spatial misalignment between datasets.  

1. The Definition of an Extreme 
After some research, I decided against a single absolute threshold (e.g., 50 mm) to avoid biasing the catalog toward wet regions and missing extremes in arid regions. Instead, let’s use percentiles defined per location. Then, there will be an event “core” with the highest precipitation and some region adjacent to it, where precipitation is not that extreme but to capture the whole storm extent. 

Note: it might be unnecessary for just top N (10) events, however might be useful for further analysis and considering different types of precipitation. Lmk what you think. 

Note 2: to avoid cataloging very low precipitation events in very dry regions, we can add a hard minimum threshold, e.g., at least 10 mm/day

Zhang, X., Alexander, L., Hegerl, G.C., Jones, P., Tank, A.K., Peterson, T.C., Trewin, B. and Zwiers, F.W., 2011. Indices for monitoring changes in extremes based on daily temperature and precipitation data. Wiley Interdisciplinary Reviews: Climate Change, 2(6), pp.851-870.

Metric: Percentile-based thresholds derived from "wet days" (days with precipitation≥1 mm, to avoid skewing the distribution with zero values).

- The "Core" threshold: 99th percentile & >= 10 mm/ day. This defines the intense center of the event.

- The extent of an event threshold: 90th percentile. This defines the full physical extent of the storm system.
  
Alexander, L.V., Zhang, X., Peterson, T.C., Caesar, J., Gleason, B., Klein Tank, A.M.G., Haylock, M., Collins, D., Trewin, B., Rahimzadeh, F. and Tagipour, A., 2006. Global observed changes in daily climate extremes of temperature and precipitation. Journal of Geophysical Research: Atmospheres, 111(D5).

- For the global standard (ETCCDI indices) of using 95th/99th percentiles for "very wet" and "extremely wet" days.

Stinnett, S.N., Gensini, V.A., Haberlie, A.M., Michaelis, A.C. and Ashley, W.S., 2024. Changes in extreme daily precipitation over the contiguous United States from Convection-Permitting simulations. Journal of Applied Meteorology and Climatology, 63(12), pp.1523-1543.
Jong, B.T., Delworth, T.L., Cooke, W.F., Tseng, K.C. and Murakami, H., 2023. Increases in extreme precipitation over the Northeast United States using high-resolution climate model simulations. Npj Climate and Atmospheric Science, 6(1), p.18.

- For the 99th percentile as the standard for monitoring extreme storms in the USA.

Note: here I am talking about annual percentiles. An alternative would be to define seasonal percentiles to account for different intensities of the extreme events for different seasons. Assuming the annual approach is more relevant to hydrologic risks (while seasonal might be more relevant to meteorological applications), I’d say to consider annual. 

2. Spatial extent
   
Using Hysteresis thresholding (2 thresholds), we’ll identify the storm core (pixels > 99th percentile), and keep moderate rain connected to the core (pixels > let’s say, 90th percentile). This will also help removing isolated moderate rain elsewhere
Here we additionally need to consider the minimum area required to be included in the catalog, taking into account high spatial resolution of the data (1 km).
Proposed value for the minimum area: ~500 km2 (22×22 pixels or 23 by 23 pixel).
Aligned with the lower bound for Meso-β scale events. 
Shou, S., Li, S., Shou, Y. and Yao, X., 2023. An introduction to mesoscale meteorology. Singapore: Springer.
Note: there might be some domain border-related artifacts - e.g., for a coastal impact that only captures a small fraction of the larger storm off shore. Option: include a domain border flag and accept a smaller minimum area if the flag is true. Do you think that should be taken into account?
Further, for more straightforward visualization and cross-dataset comparison, I propose to extract a data cube. I.e., 
(1) identify the centroid of the event, 
(2) find the maximum distance defined by the second threshold​
(3) take the data with the center at the centroid and extending in all directions according to the distance found in (2).
This will allow capturing a spatial domain that will contain the event, even if its center and/or extent is shifted in the other datasets (rather than strictly taking the mask according to the chosen shape following the defined percentiles).

4. Catalog structure
Part 1: The master index (csv)
This will contain the Cube coordinates for every event. Can be used to query any dataset.
Event_ID
Date
Lat_Center
Lon_Center
Lat_Min
Lat_Max
Lon_Min
Lon_Max
AORC_ intensity

AORC_intensity I propose to calculate as the sum of all intensity values in each > 99 percentile cells (not normalized by the number of cells) such that the total intensity incorporates the spatial extent.

Note2 – Handling multi-days events: The identification algorithm is supposed to work in 3D space, hence a multi-day event will have a unique event ID. Then, a center should be defined per day to allow moving and following the storm (and not including extra empty regions) but the spatial extent should be defined as the maximum among all days and applied to all days. In this manner, the final multiday event will have a consistent size for a rectangular multidimensional array for the whole event.  

Part 2: The event chunks 

Separately, the data will be extracted from the AORC dataset according to the proposed mask and saved as individual chunks.

