---
{"dg-publish":true,"permalink":"/code-fragments/randomly-offsetting-stacked-coordinates/","title":"Spatial Jitter: Randomly Offsetting Stacked Coordinates","tags":["code-fragment","python","gis","geospatial","mapping"]}
---

### The Problem
When mapping historical or archaeological datasets, we often only have a single, generalized location for dozens of individual records (e.g., 50 events all geocoded to the exact same building or town center). If you plot this raw data on a map, all 50 points stack directly on top of each other, looking like a single event. 

### The Solution
To make the volume of data visible without losing its geographic context, we can introduce a "spatial jitter." This function takes a central latitude/longitude point and randomly offsets it by a specified distance (in meters) and a random compass bearing. 


> [!warning] Do not go crazy with the maximum distance offset. 
> A range of 100m to 600m is usually enough to separate the points visually.
>
>  If you set the radius too large, you might end up plotting your land-based historical events right into the middle of the ocean!
> 

### Dependencies
You will need to install the `geographiclib` package, which handles the complex mathematics of moving across the curvature of the Earth.
```bash
pip install geographiclib
```

### The Code

```python
import random
from geographiclib.geodesic import Geodesic

def apply_spatial_jitter(lat, lon, min_dist_meters, max_dist_meters):
    """
    Takes a coordinate and randomly shifts it within a specified radius.
    """
    geod = Geodesic.WGS84
    
    # Generate a random compass bearing (1 to 359 degrees)
    azimuth = random.randint(1, 359)   
    
    # Generate a random distance to shift the point
    shift_distance = random.randint(min_dist_meters, max_dist_meters)                  
    
    # Calculate the new coordinate based on bearing and distance
    result = geod.Direct(lat, lon, azimuth, shift_distance)
    
    # Extract the new lat/lon from the result dictionary
    new_coord = [result['lat2'], result['lon2']]                     
    
    return new_coord
```


### Example Usage

```python
# The original center point (e.g., a colonial settlement)
original_lat = -34.9285
original_lon = 138.6007

# Apply a random jitter between 100m and 500m
new_location = apply_spatial_jitter(original_lat, original_lon, 100, 500)

print(f"Original: {original_lat}, {original_lon}")
print(f"Jittered: {new_location[0]:.4f}, {new_location[1]:.4f}")
```


### Cite this post

If you found this useful for your own work, you can cite it here:

> McLean, Mark. "The Shape of History: Why We Use the Entity-Attribute-Value (EAV) Model." *The Digital Stockade*. Published 2026-03-28. https://thedigitalstockade.github.io/code-fragments/randomly-offsetting-stacked-coordinates/