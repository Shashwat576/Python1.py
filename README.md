from flask import Flask, request, jsonify, render_template_string
from geopy.geocoders import Nominatim
from geopy.exc import GeocoderTimedOut, GeocoderServiceError
from skyfield.api import Topos, EarthSatellite, load

app = Flask(__name__)

# Manually specifying the TLE for Landsat 8
tle_lines = [
    "1 39084U 13008A   23271.23641325  .00000064  00000+0  47840-4 0  9995",
    "2 39084  98.2036 292.8002 0000985  96.4282  49.6409 14.57111910599428"
]

landsat = EarthSatellite(tle_lines[0], tle_lines[1], 'LANDSAT 8', load.timescale())

@app.route('/')
def home():
    return '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>Landsat Overpass Timing App</title>
        <style>
            body {
                margin: 0;
                padding: 0;
                font-family: Arial, sans-serif;
                background: linear-gradient(135deg, #71b7e6, #9b59b6);
                height: 100vh;
                display: flex;
                justify-content: center;
                align-items: center;
                flex-direction: column;
            }
            #map {
                height: 50vh;
                width: 80vw;
                margin: 20px auto;
                border-radius: 10px;
                border: 2px solid #9b59b6;
            }
            .container {
                background-color: white;
                padding: 30px;
                border-radius: 10px;
                box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
                text-align: center;
                margin-bottom: 20px;
            }
            input[type="text"] {
                width: 80%;
                padding: 10px;
                border: 2px solid #9b59b6;
                border-radius: 5px;
                margin-bottom: 20px;
                font-size: 14px;
            }
            button {
                background-color: #9b59b6;
                color: white;
                padding: 10px 20px;
                border: none;
                border-radius: 5px;
                cursor: pointer;
                font-size: 16px;
            }
            button:hover {
                background-color: #8e44ad;
            }
        </style>
        <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
        <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
    </head>
    <body>
        <div class="container">
            <h1>Welcome to the Landsat Overpass Timing App!</h1>
            <form action="/get_overpass_location" method="get">
                <label for="location">Enter a location name:</label>
                <input type="text" id="location" name="location" required>
                <button type="submit">Get Overpass Time by Location</button>
            </form>
        </div>
        <div id="map"></div>

        <script>
            var map = L.map('map').setView([20.5937, 78.9629], 5); // Centered at India
            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                maxZoom: 18,
            }).addTo(map);

            function onMapClick(e) {
                var lat = e.latlng.lat;
                var lon = e.latlng.lng;
                
                fetch(`/get_overpass?lat=${lat}&lon=${lon}`)
                    .then(response => response.json())
                    .then(data => {
                        if (data.error) {
                            alert(data.error);
                        } else {
                            alert(`Latitude: ${data.latitude}, Longitude: ${data.longitude}, Next Overpass Time: ${data.next_overpass_time}`);
                        }
                    })
                    .catch(error => console.error('Error:', error));
            }

            map.on('click', onMapClick);
        </script>
    </body>
    </html>
    '''

# Route for clicking on the map to get overpass timing
@app.route('/get_overpass', methods=['GET'])
def get_overpass():
    lat = request.args.get('lat')
    lon = request.args.get('lon')

    try:
        lat = float(lat)
        lon = float(lon)
        
        observer_location = Topos(latitude_degrees=lat, longitude_degrees=lon)
        ts = load.timescale()
        t0 = ts.now()
        t1 = ts.now() + 1  # Set to check for the next overpass within 1 day

        times, events = landsat.find_events(observer_location, t0, t1, altitude_degrees=10.0)

        for ti, event in zip(times, events):
            if event == 0:  # Event 0 corresponds to 'rise' (satellite coming into view)
                return jsonify({
                    'latitude': lat,
                    'longitude': lon,
                    'next_overpass_time': ti.utc_iso()
                })
        return jsonify({'error': 'No overpass found in the next day!'}), 404

    except ValueError:
        return jsonify({'error': 'Invalid latitude or longitude!'}), 400
    except Exception as e:
        return jsonify({'error': str(e)}), 500

# Route for manual location name input to get overpass timing
@app.route('/get_overpass_location', methods=['GET'])
def get_overpass_location():
    location_name = request.args.get('location')
    geolocator = Nominatim(user_agent="geoapiExercises")

    try:
        location = geolocator.geocode(location_name, timeout=10)
        
        if location:
            lat = location.latitude
            lon = location.longitude
            
            observer_location = Topos(latitude_degrees=lat, longitude_degrees=lon)
            ts = load.timescale()
            t0 = ts.now()
            t1 = ts.now() + 1

            times, events = landsat.find_events(observer_location, t0, t1, altitude_degrees=10.0)
            
            for ti, event in zip(times, events):
                if event == 0:
                    return render_template_string('''
                    <!DOCTYPE html>
                    <html>
                    <head>
                        <title>Landsat Overpass Result</title>
                        <style>
                            body {
                                margin: 0;
                                padding: 0;
                                font-family: Arial, sans-serif;
                                background: linear-gradient(135deg, #71b7e6, #9b59b6);
                                height: 100vh;
                                display: flex;
                                justify-content: center;
                                align-items: center;
                            }
                            .result-container {
                                background-color: white;
                                padding: 30px;
                                border-radius: 10px;
                                box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
                                text-align: center;
                            }
                            h2 {
                                color: #333;
                                font-size: 24px;
                                margin-bottom: 20px;
                            }
                            p {
                                font-size: 18px;
                                color: #555;
                            }
                        </style>
                    </head>
                    <body>
                        <div class="result-container">
                            <h2>Overpass Information</h2>
                            <p><strong>Latitude:</strong> {{ latitude }}</p>
                            <p><strong>Longitude:</strong> {{ longitude }}</p>
                            <p><strong>Next Overpass Time:</strong> {{ next_overpass_time }}</p>
                            <a href="/" style="text-decoration: none; color: #9b59b6; font-weight: bold;">Back to Home</a>
                        </div>
                    </body>
                    </html>
                    ''', latitude=lat, longitude=lon, next_overpass_time=ti.utc_iso())
            
            return jsonify({'error': 'No overpass found in the next day!'}), 404
        else:
            return jsonify({'error': 'Location not found!'}), 404

    except GeocoderTimedOut:
        return jsonify({'error': 'The geocoding service timed out. Please try again later.'}), 504
    except GeocoderServiceError:
        return jsonify({'error': 'There was an issue with the geocoding service. Please try again.'}), 500

if __name__ == "__main__":
    app.run(debug=True, use_reloader=False)
 * Serving Flask app "__main__" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 
