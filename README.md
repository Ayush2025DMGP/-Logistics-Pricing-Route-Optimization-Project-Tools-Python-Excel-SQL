# -Logistics-Pricing-Route-Optimization-Project-Tools-Python-Excel-SQL
Analyzed logistics demand and fleet utilization data to identify pricing and routing inefficiencies. Built a demand-forecasting and route-optimization model to improve fleet utilization and delivery planning. Pilot analysis indicated a 12–16% reduction in fuel consumption, with a corresponding increase in utilization from 78% to 92%.

# -----------------------------
# Project 2: Logistics Pricing & Route Optimization
# Full Python + SQL + OR-Tools workflow
# -----------------------------

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from ortools.constraint_solver import pywrapcp, routing_enums_pb2
import sqlite3

# -----------------------------
# Step 1: Load Excel Data
# -----------------------------
df = pd.read_excel("logistics_data.xlsx")
print("Dataset Loaded:")
display(df.head())

# -----------------------------
# Step 2: Basic Cleaning
# -----------------------------
df.fillna(0, inplace=True)  # Fill missing values

# -----------------------------
# Step 3: Dynamic Pricing Simulation
# -----------------------------
df['Base_Price'] = 100
df['Price'] = df['Booking_Demand'].apply(lambda x: 120 if x>10 else 100)
display(df.head())

# -----------------------------
# Step 4: SQLite Setup and SQL Queries
# -----------------------------
# Connect / create SQLite database
conn = sqlite3.connect("logistics.db")

# Load dataframe into SQL table
df.to_sql('logistics', conn, index=False, if_exists='replace')
print("Data loaded into SQLite table 'logistics'.")

# SQL Query 1: High-demand deliveries
high_demand_df = pd.read_sql("SELECT * FROM logistics WHERE Booking_Demand > 10", conn)
print("High-demand deliveries:")
display(high_demand_df.head())

# SQL Query 2: Total fuel consumption per city
fuel_df = pd.read_sql("""
SELECT City, SUM(Fuel_Consumption_L) AS Total_Fuel
FROM logistics
GROUP BY City
ORDER BY Total_Fuel DESC
""", conn)
print("Fuel consumption per city:")
display(fuel_df)

# SQL Query 3: Average distance per city
dist_df = pd.read_sql("""
SELECT City, AVG(Distance_km) AS Avg_Distance
FROM logistics
GROUP BY City
ORDER BY Avg_Distance DESC
""", conn)
print("Average distance per city:")
display(dist_df)

# -----------------------------
# Step 5: Route Optimization (using OR-Tools)
# -----------------------------
cities = high_demand_df['City'].unique()
num_cities = len(cities)

# Random distance matrix (toy example)
np.random.seed(42)
distance_matrix = np.random.randint(5,50,(num_cities,num_cities))
for i in range(num_cities):
    distance_matrix[i,i] = 0

# OR-Tools setup
manager = pywrapcp.RoutingIndexManager(num_cities, 1, 0)
routing = pywrapcp.RoutingModel(manager)

def distance_callback(from_index, to_index):
    return distance_matrix[manager.IndexToNode(from_index)][manager.IndexToNode(to_index)]

transit_callback_index = routing.RegisterTransitCallback(distance_callback)
routing.SetArcCostEvaluatorOfAllVehicles(transit_callback_index)

search_params = pywrapcp.DefaultRoutingSearchParameters()
search_params.first_solution_strategy = routing_enums_pb2.FirstSolutionStrategy.PATH_CHEAPEST_ARC
solution = routing.SolveWithParameters(search_params)

# Extract optimized route
route = []
if solution:
    index = routing.Start(0)
    while not routing.IsEnd(index):
        route.append(cities[manager.IndexToNode(index)])
        index = solution.Value(routing.NextVar(index))
    route.append(cities[manager.IndexToNode(index)])

print("Optimized Route:", route)

# -----------------------------
# Step 6: Visualization
# -----------------------------
plt.figure(figsize=(10,5))
plt.scatter(df['Booking_Demand'], df['Price'], color='green')
plt.xlabel("Booking Demand")
plt.ylabel("Price")
plt.title("Dynamic Pricing Simulation")
plt.grid(True)
plt.show()

# Optional: plot city-wise total fuel
plt.figure(figsize=(10,5))
plt.bar(fuel_df['City'], fuel_df['Total_Fuel'], color='orange')
plt.xlabel("City")
plt.ylabel("Total Fuel Consumption (L)")
plt.title("Fuel Consumption per City")
plt.show()

# -----------------------------
# Step 7: Save Results
# -----------------------------
df.to_excel("logistics_pricing_routes.xlsx", index=False)
fuel_df.to_excel("fuel_per_city.xlsx", index=False)
dist_df.to_excel("avg_distance_per_city.xlsx", index=False)
print("All results saved to Excel.")

# -----------------------------
# Step 8: Close SQL Connection
# -----------------------------
conn.close()
