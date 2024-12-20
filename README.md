# CET4973-Final-Project
import requests
import folium
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
from folium.plugins import HeatMap
from geopy.distance import geodesic
from IPython.display import display, IFrame

# Function to fetch real-time traffic data
def fetch_traffic_data(api_key, bbox):
    url = "https://api.tomtom.com/traffic/services/5/incidentDetails"
    params = {'key': api_key, 'bbox': bbox, 'language': 'en-US'}
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        print(f"Error fetching traffic data: {response.status_code}")
        return None

# Define the central point and bounding box
center_coords = (40.693, -73.987)  #Central point in Brooklyn, NY
bbox = "-74.000,40.690,-73.975,40.705"  # Bounding box Jay St Metrotech are
api_key = "BX8Auximt0noAQIlxo1x35DBGy6H1GhB"  

# Fetch traffic data
data = fetch_traffic_data(api_key, bbox)

# Parse the data if it exists
if data and 'incidents' in data:
    traffic_data = []
    for incident in data['incidents']:
        geometry = incident.get('geometry', {})
        properties = incident.get('properties', {})
        coordinates = geometry.get('coordinates', [])
        icon_category = properties.get('iconCategory', 0)  # Use iconCategory as a proxy for traffic level

        # Extract latitude and longitude from coordinates
        for coord in coordinates:
            lat, lon = coord[1], coord[0]
            distance = geodesic(center_coords, (lat, lon)).meters  # Calculate distance
            if distance <= 400:  # Include only incidents within 5 blocks (~400 meters)
                traffic_data.append({
                    "latitude": lat,
                    "longitude": lon,
                    "predicted_traffic": icon_category
                })

    # Convert to DataFrame for analysis
    df = pd.DataFrame(traffic_data)

    # Check if DataFrame has data
    if df.empty:
        print("No traffic incidents to display within a 5-block radius.")
    else:
        # Create a folium map
        map_center = [df['latitude'].mean(), df['longitude'].mean()]
        my_map = folium.Map(location=map_center, zoom_start=15)

        # Add heatmap
        heatmap_data = [[row['latitude'], row['longitude'], row['predicted_traffic']] for _, row in df.iterrows()]
        HeatMap(heatmap_data).add_to(my_map)

        # Add markers for incidents
        for _, row in df.iterrows():
            folium.Marker(
                [row['latitude'], row['longitude']],
                popup=f"Predicted Traffic: {row['predicted_traffic']}"
            ).add_to(my_map)

        # Display map inline in Colab
        display(my_map)

        # Visualizations
        plt.figure(figsize=(15, 12))

        # Bar Plot for Traffic by Latitude
        plt.subplot(2, 2, 1)
        sns.barplot(x='latitude', y='predicted_traffic', data=df,)
        plt.title('Predicted Traffic Volume by Location (Latitude)')
        plt.xticks(rotation=45)

        # Scatter Plot for Traffic Incidents
        plt.subplot(2, 2, 2)
        sns.scatterplot(x='longitude', y='latitude', hue='predicted_traffic', size='predicted_traffic', sizes=(20, 200), data=df, palette='coolwarm', legend=False)
        plt.title('Traffic Incidents (Traffic Volume vs Location)')

        # Line Plot for Traffic Volume Over Time
        plt.subplot(2, 2, 3)
        sns.lineplot(x=df.index, y='predicted_traffic', data=df, marker='o', color='green')
        plt.title('Traffic Volume Over Time')


        plt.tight_layout()
        plt.show()

else:
    print("No traffic data found.")
