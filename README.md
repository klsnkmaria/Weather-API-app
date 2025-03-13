import datetime as dt
import json
import requests
import sys
from flask import Flask, jsonify, request


API_TOKEN = ""

WEATHER_API_KEY = ""

app = Flask(__name__)


class InvalidUsage(Exception):
    status_code = 400

    def __init__(self, message, status_code=None, payload=None):
        Exception.__init__(self)
        self.message = message
        if status_code is not None:
            self.status_code = status_code
        self.payload = payload

    def to_dict(self):
        rv = dict(self.payload or ())
        rv["message"] = self.message
        return rv


@app.errorhandler(InvalidUsage)
def handle_invalid_usage(error):
    response = jsonify(error.to_dict())
    response.status_code = error.status_code
    return response


def get_weather(location, date):
    url = f"https://weather.visualcrossing.com/VisualCrossingWebServices/rest/services/timeline/{location}?unitGroup=us&key=4RSPJWK6FL6HLHE6L9JJPMR9A&contentType=json"
    response = requests.get(url)

    if response.status_code != 200:
        print('Unexpected Status code: ', response.status_code)
        sys.exit()

    jsonData = response.json()

    if "days" not in jsonData:
        raise InvalidUsage("No weather data available for this date", status_code=404)

    weather = next((day for day in jsonData["days"] if day["datetime"] == date), None)
    if not weather:
        raise InvalidUsage("No weather data found for the requested date", status_code=404)

    return {
        "temp_c": (weather["temp"] - 32) * 5 / 9,  # Convert to Celsius
        "wind_kph": weather["windspeed"] * 1.60934,  # Convert mph to kph
        "pressure_mb": weather.get("pressure", "N/A"),
        "humidity": weather["humidity"],
        "condition": weather["conditions"]
    }


@app.route("/weather", methods=["POST"])
def weather_endpoint():
    start_dt = dt.datetime.utcnow()
    json_data = request.get_json()

    if not json_data.get("token"):
        raise InvalidUsage("token is required", status_code=400)
    if json_data.get("token") != API_TOKEN:
        raise InvalidUsage("wrong API token", status_code=403)
    if not json_data.get("requester_name"):
        raise InvalidUsage("requester_name is required", status_code=400)
    if not json_data.get("location"):
        raise InvalidUsage("location is required", status_code=400)
    if not json_data.get("date"):
        raise InvalidUsage("date is required", status_code=400)

    location = json_data.get("location")
    date = json_data.get("date")

    weather_info = get_weather(location, date)

    end_dt = dt.datetime.utcnow()

    result = {
        "requester_name": json_data["requester_name"],
        "timestamp": end_dt.isoformat() + "Z",
        "location": location,
        "date": date,
        "weather": weather_info
    }

    return jsonify(result)


@app.route("/")
def home_page():
    return "<p><h2>Weather API Service</h2></p>"


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
