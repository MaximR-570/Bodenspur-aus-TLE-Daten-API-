# Bodenspur-aus-TLE-Daten-API-
Darstellung der Bodenspur von Satelliten auf Basis von TLE-Daten, die von einer öffentlich zugänglichen API eingelesen werden.

import requests
from skyfield.api import EarthSatellite, load, wgs84
import matplotlib.pyplot as plt
import numpy as np
import cartopy.crs as ccrs 
import cartopy.feature as cfeature
import matplotlib.path as mpath

#%%
#
# Auslesen der TLE-Daten aus der API
#
# ISS = 25544
# Hubble = 20580
# Molniya = 41032
# Sonnensynchron = 67377
# Geostationär = 28786
# 

url = "https://tle.ivanstanojevic.me/api/tle/25544"

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,application/json,*/*;q=0.8",
    "Accept-Language": "de-DE,de;q=0.9,en-US;q=0.8,en;q=0.7",
    "Connection": "close"}

try:
    response = requests.get(url, headers=headers, timeout=10)

    # print(response.json())

    data = response.json()
    line1 = data['line1']
    line2 = data['line2']
    
except requests.exceptions.RequestException as exception:
    print(f"Fehler bei der Abfrage: {exception}")
    
    
#%% 
#
# Definieren von Parametern und Berechnung der Bodenspur-Koordinaten
#

GS_LAT = 53.6
GS_LON = 8.5
   
satellite = EarthSatellite(line1, line2, "Satellit")  
umlaufzeit = 2 * np.pi / satellite.model.no_kozai

Dauer_Propagation = umlaufzeit / 60 # Funktioniert für Molniya-Umlauf nur unvollständig
Zeitschritt = 5

ts = load.timescale()
t0 = satellite.epoch

N = (Dauer_Propagation * 3600 / Zeitschritt) + 1
times = ts.tt_jd(t0.tt + np.arange(N) * Zeitschritt / 86400)
position = satellite.at(times)

subpoint = wgs84.subpoint(position)
lat = subpoint.latitude.degrees
lon = subpoint.longitude.degrees

sprung = np.abs(np.diff(lon)) > 180 
lon[1:][sprung] = np.nan


#%%
# 
# Bodenspur auf Weltkarte darstellen
#

fig = plt.figure(figsize = (16,8))
ax = plt.axes(projection = ccrs.PlateCarree())
ax.coastlines()
ax.set_global()
ax.gridlines(linewidth = 0.5, linestyle = "--")
plt.plot(lon, lat, color = 'blue')
plt.show()

#%%
# Polarplot
#
# Bodenspur im Polarplot darstellen
#
'''
umlaufzeit_tage = umlaufzeit / 1440.0
t_start = times[0]
times_1_orbit = times[times.tt <= (t_start.tt + umlaufzeit_tage)]

geocentric = satellite.at(times_1_orbit)
lat, lon = wgs84.latlon_of(geocentric)

lat_deg = lat.degrees
lon_deg = lon.degrees

fig = plt.figure()
ax = fig.add_subplot(111, polar=True)

ax.plot(np.radians(lon_deg), 90 - lat_deg, label="Orbit")
ax.plot(np.radians(GS_LON), 90 - GS_LAT, 'ro', label="Bodenstation")

ax.set_ylim(0, 90)
ax.set_rticks([15, 30, 45, 60, 75, 90])
ax.set_yticklabels(["75°N", "60°N", "45°N", "30°N", "15°N", "0°"])
ax.set_theta_zero_location("S")
ax.legend(loc="upper right")
plt.show()
'''

#%%


GS_LAT = 54.12
GS_LON = 9.6

umlaufzeit_min = (2 * np.pi) / satellite.model.no_kozai
umlaufzeit_tage = umlaufzeit_min / 1440.0

t_start = times[0]
times_1_orbit = times[times.tt <= (t_start.tt + umlaufzeit_tage)]

geocentric = satellite.at(times_1_orbit)
lat, lon = wgs84.latlon_of(geocentric)
lat_deg = lat.degrees
lon_deg = lon.degrees

fig = plt.figure(figsize=(8, 8))
ax = fig.add_subplot(1, 1, 1, projection=ccrs.NorthPolarStereo())

ax.set_extent([-180, 180, 0, 90], crs=ccrs.PlateCarree())

theta = np.linspace(0, 2 * np.pi, 100)
center, radius = [0.5, 0.5], 0.5
verts = np.vstack([np.sin(theta), np.cos(theta)]).T
circle = mpath.Path(verts * radius + center)
ax.set_boundary(circle, transform=ax.transAxes)

ax.add_feature(cfeature.COASTLINE)

ax.plot(lon_deg, lat_deg, color="blue", linewidth=1.5,
        transform=ccrs.PlateCarree(), label="Aktueller Orbit")

ax.plot(GS_LON, GS_LAT, "ro", markersize=4,
        transform=ccrs.PlateCarree(), label="Heide")

ax.legend(loc="upper right")

plt.title("Bodenspur auf der Nordhalbkugel")
plt.show()

