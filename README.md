# 49ER_DATA_ANALYSIS
Logiciel de traitement de données sous format GPX

import gpxpy
import pandas as pd
import folium
import plotly.graph_objects as go
from plotly.subplots import make_subplots
import os
import webbrowser

# Charger le fichier GPX
def load_gpx(file_path):
    with open(file_path, 'r') as gpx_file:
        gpx = gpxpy.parse(gpx_file)
    return gpx

# Extraire les données des traces
def extract_gpx_data(gpx):
    data = {
        "latitude": [],
        "longitude": [],
        "elevation": [],
        "time": [],
        "distance": [0]  # Distance cumulée
    }
    for track in gpx.tracks:
        for segment in track.segments:
            prev_point = None
            for point in segment.points:
                # Ajouter une condition pour ignorer les points avec des valeurs manquantes
                if point.latitude is None or point.longitude is None or point.time is None:
                    continue
                data["latitude"].append(point.latitude)
                data["longitude"].append(point.longitude)
                data["elevation"].append(point.elevation)
                data["time"].append(point.time)
                
                # Calculer la distance cumulée
                if prev_point is not None:
                    distance = point.distance_2d(prev_point)
                    data["distance"].append(data["distance"][-1] + distance)
                prev_point = point
    
    df = pd.DataFrame(data)
    df['time'] = pd.to_datetime(df['time'])  # Convertir en datetime
    return df

# Calculer la vitesse en km/h et en nœuds
def calculate_speed(df):
    df['time_diff'] = df['time'].diff().dt.total_seconds() / 3600  # en heures
    df['distance_diff'] = df['distance'].diff() / 1000  # en km
    df['speed'] = df['distance_diff'] / df['time_diff']
    df['speed'] = df['speed'].fillna(0)  # Remplacer les NaN par 0

    # Conversion de la vitesse en nœuds
    df['speed_knots'] = df['speed'] * 0.5399568
    return df

# Visualiser les données sur une carte avec Folium
def plot_map(df, start_time, end_time):
    # Filtrer les données en fonction de l'intervalle de temps sélectionné
    filtered_df = df[(df['time'] >= start_time) & (df['time'] <= end_time)]
    
    # Si aucune donnée n'est disponible pour cet intervalle, retourner une carte vide
    if filtered_df.empty:
        print("Aucune donnée pour cet intervalle de temps.")
        return folium.Map(location=[df['latitude'].mean(), df['longitude'].mean()], zoom_start=13)

    # Créer une carte centrée sur l'intervalle sélectionné
    map_center = [filtered_df['latitude'].mean(), filtered_df['longitude'].mean()]
    gpx_map = folium.Map(location=map_center, zoom_start=13)

    # Tracer uniquement la portion filtrée de la trace
    coordinates = list(zip(filtered_df['latitude'], filtered_df['longitude']))
    folium.PolyLine(coordinates, color="blue", weight=2.5, opacity=1).add_to(gpx_map)
    
    return gpx_map

# Graphiques interactifs avec Plotly
def plot_segment_analysis(df, start_time, end_time):
    # Filtrer les données pour l'intervalle sélectionné
    segment = df[(df['time'] >= start_time) & (df['time'] <= end_time)]
    
    # Créer un subplot avec deux graphiques
    fig = make_subplots(rows=2, cols=1, shared_xaxes=True, vertical_spacing=0.1)

    # Graphique 1: Vitesse en fonction de la distance (en nœuds)
    fig.add_trace(
        go.Scatter(x=segment['distance'], y=segment['speed_knots'], mode='lines', name='Vitesse (nœuds)', line=dict(color='blue')),
        row=1, col=1
    )
    fig.update_yaxes(title_text="Vitesse (nœuds)", row=1, col=1)

    # Graphique 2: Altitude en fonction de la distance
    fig.add_trace(
        go.Scatter(x=segment['distance'], y=segment['elevation'], mode='lines', name='Altitude (m)', line=dict(color='green')),
        row=2, col=1
    )
    fig.update_yaxes(title_text="Altitude (m)", row=2, col=1)

    # Mise en forme générale
    fig.update_layout(title_text="Analyse du segment", height=600, showlegend=False)
    
    return fig.to_html(full_html=False)

# Générer et sauvegarder la carte et les graphiques dans un fichier HTML
# Sauvegarder le fichier HTML avec l'encodage utf-8
def save_map_with_graph(df, start_time, end_time, file_name="map_with_graphs.html"):
    # Créer la carte avec Folium
    gpx_map = plot_map(df, start_time, end_time)
    map_html = gpx_map._repr_html_()  # Convertir la carte en HTML

    # Créer les graphiques avec Plotly
    plot_html = plot_segment_analysis(df, start_time, end_time)

    # Combiner la carte et les graphiques dans un fichier HTML
    html_content = f"""
    <html>
    <head>
        <title>Trace GPX et Analyse</title>
        <style>
            body {{
                font-family: Arial, sans-serif;
            }}
            .container {{
                display: flex;
                flex-direction: column;
                align-items: center;
                margin-top: 20px;
            }}
            .map-container {{
                width: 80%;
                height: 600px;
                margin-bottom: 20px;
            }}
            .graphs-container {{
                width: 80%;
                margin-top: 20px;
            }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Carte et Analyse de la Trace GPX</h1>
            
            <div class="map-container">
                {map_html}  <!-- Carte Folium -->
            </div>
            
            <div class="graphs-container">
                {plot_html}  <!-- Graphiques Plotly -->
            </div>
        </div>
    </body>
    </html>
    """
    
    # Sauvegarder le fichier HTML avec l'encodage utf-8
    with open(file_name, "w", encoding="utf-8") as f:
        f.write(html_content)
    
    print(f"Fichier HTML généré : {file_name}")
    webbrowser.open(file_name)

# Main
if __name__ == "__main__":
    gpx_file = "C:\\Users\\albe2\\Desktop\\49er\\NAV07.gpx"  # Remplacez par votre fichier GPX
    
    # Charger et traiter les données GPX
    gpx = load_gpx(gpx_file)
    df = extract_gpx_data(gpx)
    df = calculate_speed(df)

    # Générer la carte et les graphiques dans un fichier HTML
    save_map_with_graph(df, df['time'].min(), df['time'].max())
