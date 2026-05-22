import requests
import time
from geopy.geocoders import Nominatim
from typing import Dict, Any

#API keys
NREL_API_KEY = "YOUR_NREL_API_KEY_HERE"
TOMTOM_API_KEY = "YOUR_TOMTOM_API_KEY_HERE"

ELEVATION_API = "https://api.open-elevation.com/api/v1/lookup"
WEATHER_API = "https://api.open-meteo.com/v1/forecast"

def get_country_code(country: str) -> str: #remove need to input both country and code
    country_map = {
        "canada": "CA",
        "united states": "US",
        "usa": "US",
        "mexico": "MX"
    }

    return country_map.get(country.lower())

#geolocation finder (convert city to coordinates for use with APIs)
def get_location(city, country):
    geolocator = Nominatim(user_agent="ev_super_app")
    location = geolocator.geocode(f"{city}, {country}")

    if not location:
        return None

    return {
        "lat": location.latitude,
        "lon": location.longitude,
        "address": location.address,
        "bbox": location.raw.get("boundingbox", None)
    }

#EV station functions
def get_ev_data(city: str, country_code: str):
    url = "https://developer.nrel.gov/api/alt-fuel-stations/v1.json"

    params = {
        "api_key": NREL_API_KEY,
        "fuel_type": "ELEC",
        "city": city,
        "country": country_code.upper(),
        "status": "E",
        "access": "public",
        "limit": "all"
    }

    try:
        res = requests.get(url, params=params)
        data = res.json()

        stations = data.get("fuel_stations", [])

        city_stations = [
            s for s in stations
            if s.get("city", "").lower() == city.lower()
        ]

        total = len(city_stations)
        dc_fast = len([s for s in city_stations if s.get("ev_dc_fast_num")])
        level2 = len([s for s in city_stations if s.get("ev_level2_evse_num")])

        print("\n--- EV Charging Infrastructure ---")
        print(f"Stations: {total}")
        print(f"DC Fast: {dc_fast}")
        print(f"Level 2: {level2}")

        return city_stations   

    except Exception as e:
        print("EV API Error:", e)
        return []             

#elevation functions
def get_elevation(lat, lon):
    payload = {"locations": [{"latitude": lat, "longitude": lon}]}

    try:
        res = requests.post(ELEVATION_API, json=payload, timeout=10)
        res.raise_for_status()
        return res.json()["results"][0]["elevation"]
    except:
        return None

#weather functions
def get_weather(lat, lon):
    params = {
        "latitude": lat,
        "longitude": lon,
        "current": [
            "temperature_2m",
            "relative_humidity_2m",
            "windspeed_10m",
            "precipitation"
        ]
    }

    try:
        res = requests.get(WEATHER_API, params=params)
        data = res.json()["current"]

       
        print("\n--- Weather Conditions ---")
        print(f"Temperature: {data['temperature_2m']} °C")
        print(f"Humidity: {data['relative_humidity_2m']} %")
        print(f"Wind Speed: {data['windspeed_10m']} km/h")
        print(f"Precipitation: {data['precipitation']} mm")

       
        return {
            "temperature_avg": data["temperature_2m"]
        }

    except:
        print("Weather API Error")
        return {"temperature_avg": 15}
    
def get_optimal_ev_speed(frc: str) -> int:
    return 90 if frc in ['FRC0', 'FRC1', 'FRC2'] else 40

def get_tomtom_flow(lat: float, lon: float):
    url = "https://api.tomtom.com/traffic/services/4/flowSegmentData/absolute/18/json"
    params = {"key": TOMTOM_API_KEY, "point": f"{lat},{lon}", "unit": "kmph"}

    try:
        res = requests.get(url, params=params, timeout=10)

        if res.status_code != 200:
            return None

        data = res.json()

        if not data or "flowSegmentData" not in data:
            return None

        flow = data["flowSegmentData"]

        required_keys = [
            "currentSpeed",
            "freeFlowSpeed",
            "currentTravelTime",
            "freeFlowTravelTime",
            "frc"
        ]
        if not all(key in flow for key in required_keys):
            return None

        curr = flow["currentSpeed"]
        free = flow["freeFlowSpeed"]

        if free == 0:
            return None

        delay = max(0, flow["currentTravelTime"] - flow["freeFlowTravelTime"])
        opt = get_optimal_ev_speed(flow["frc"])

        impact = "OPTIMAL"
        if curr > opt + 20:
            impact = "DRAG LOSS"
        elif curr < opt - 20:
            impact = "IDLE LOSS"

        return curr, opt, delay, impact

    except:
        return None
    
def analyze_traffic(bbox):
    if not bbox:
        print("\nNo bounding box for traffic analysis.")
        return {"congestion_ratio": 1.0}

    south, north, west, east = map(float, bbox)

    print("\n--- Traffic EV Impact Scan ---")
    print(f"{'COORD':<20} | {'SPD':<5} | {'OPT':<5} | {'DELAY':<5} | IMPACT")

    grid = 4
    lats = [south + (north - south) * i / (grid - 1) for i in range(grid)]
    lons = [west + (east - west) * i / (grid - 1) for i in range(grid)]

    ratios = []

    for lat in lats:
        for lon in lons:
            data = get_tomtom_flow(lat, lon)
            if data:
                s, o, d, imp = data
                print(f"{round(lat,3)},{round(lon,3):<10} | {s:<5} | {o:<5} | {d:<5} | {imp}")

                if o > 0:
                    ratios.append(s / o)

            time.sleep(0.2)

    avg_ratio = sum(ratios) / len(ratios) if ratios else 1.0

    return {"congestion_ratio": avg_ratio}
#MAIN FUNCTION <------
def main():
    print("=== EV Smart City Analyzer ===\n")

    city = input("Enter city: ").strip()
    country = input("Enter country (e.g., Canada): ").strip()
    country_code = get_country_code(country)

    if not country_code:
        print("Unsupported country. Please enter a valid one.")
        return

    loc = get_location(city, country)

    if not loc:
        print("Location not found.")
        return

    lat, lon = loc["lat"], loc["lon"]

    print(f"\nLocation Found: {loc['address']}")
    print(f"Coordinates: {lat}, {lon}")

    
    station_data = get_ev_data(city, country_code)

    
    elevation = get_elevation(lat, lon)
    if elevation is not None:
        print("\n--- Elevation ---")
        print(f"{elevation} meters")

        if elevation > 1000:
            print("High elevation → higher EV energy usage")
        elif elevation > 500:
            print("Moderate elevation → moderate EV energy usage")
        else:
            print("Low elevation → efficient EV usage")

    
    weather_data = get_weather(lat, lon)

   
    traffic_data = analyze_traffic(loc["bbox"])

    analyzer = EVAnalyzer(weather_data, elevation or 0, station_data, traffic_data)
    analyzer.report()

class EVAnalyzer:
    def __init__(self, weather_data, elevation, station_data, traffic_data):
        self.weather = weather_data
        self.elevation = elevation
        self.stations = station_data
        self.traffic = traffic_data

    def analyze_temperature(self):
        """Cold weather reduces EV battery efficiency"""
        temp = self.weather.get("temperature_avg", 15)

        if temp < -10:
            score = 2
        elif temp < 0:
            score = 5
        elif temp < 15:
            score = 8
        elif temp < 25:
            score = 10
        else:
            score = 5

        return {
            "factor": "Temperature",
            "score": score,
            "impact": "Extreme temperatures reduce efficiency; moderate is best"
        }

    def analyze_elevation(self):
        """Higher elevation can slightly reduce efficiency; lower is best"""
        if self.elevation > 2000:
            score = 3
        elif self.elevation > 1000:
            score = 5
        elif self.elevation > 500:
            score = 6.5
        elif self.elevation > 100:
            score = 8
        elif self.elevation >= 0:
            score = 10
        else:
            score = 5

        return {
            "factor": "Elevation",
            "score": score,
            "impact": "Higher elevations may slightly reduce efficiency"
        }

    def analyze_infrastructure(self):
        """Charging availability is critical"""
        num_stations = len(self.stations) if self.stations else 0

        if num_stations < 50:
            score = 1
        elif num_stations < 0:
            score = 0
        elif num_stations < 200:
            score = 3
        elif num_stations < 500:
            score = 4
        elif num_stations < 1000:
            score = 7
        elif num_stations > 1000:
            score = 10
        else:
            score = 5

        return {
            "factor": "Charging Infrastructure",
            "score": score,
            "impact": "More stations = easier long-distance travel"
        }

    def overall_score(self):
        factors = [
            self.analyze_temperature(),
            self.analyze_elevation(),
            self.analyze_infrastructure(),
            self.analyze_traffic()
        ]

        total = sum(f["score"] for f in factors)
        avg_score = total / len(factors)

        return avg_score, factors

    def report(self):
        avg_score, factors = self.overall_score()

        print("\n⚡ EV Suitability Report")
        print(f"Overall Score: {avg_score:.1f} / 10\n")

        for f in factors:
            print(f"{f['factor']}: {f['score']}/10")
            print(f"  → {f['impact']}\n")

        if avg_score > 8:
            print("✅ Excellent location for EV ownership")
        elif avg_score > 6:
            print("👍 Good location for EV ownership")
        elif avg_score > 4:
            print("⚠️ Moderate — consider limitations")
        else:
            print("❌ Challenging location for EV ownership")
        
    def analyze_traffic(self):

        congestion = self.traffic.get("congestion_ratio", 1.0)

        if congestion > 2.0:
            score = 4
            impact = "Severe congestion increases energy use over time and reduces convenience"
        elif congestion > 1.5:
            score = 7
            impact = "Moderate traffic; EV benefits from regen braking but longer commute times"
        else:
            score = 9
            impact = "Light traffic; EV operates efficiently"

        return {
        "factor": "Traffic",
        "score": score,
        "impact": impact
        }

if __name__ == "__main__":
    main()

    #Optional demo/testing code
    weather_data = {"temperature_avg": -5}
    elevation = 251
    stations = ["station1", "station2", "station3", "station4"]

    analyzer = EVAnalyzer(
        weather_data,
        elevation,
        stations,
        traffic_data={"congestion_ratio": 1.8}
    )

