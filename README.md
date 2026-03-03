# Radar-based-Detection-
It will help us to find obstacles and enemies get into our radar based zone , in submarines and automatically it will find safest route towards the destination point , and also help to eradicate obstacles from the path.
import pygame
import sys
import heapq
import random
import math
import time
from collections import deque

pygame.init()
info = pygame.display.Info()
WIDTH, HEIGHT = info.current_w, info.current_h
screen = pygame.display.set_mode((WIDTH, HEIGHT), pygame.FULLSCREEN)
pygame.display.set_caption("Radar Path Planning Simulator")
clock = pygame.time.Clock()
font = pygame.font.Font(None, 32)
small_font = pygame.font.Font(None, 22)
tiny_font = pygame.font.Font(None, 18)

GRID = 20
CELL = min(WIDTH - 550, HEIGHT - 100) // GRID
START_STATE_1 = (1, 1)
START_STATE_2 = (1, 3)
GOAL_STATE = (18, 18)

class IoTSensor:
    def __init__(self):
        self.detected_enemies = []
        self.scan_radius = 5
        self.detection_log = []
        
    def scan_area(self, sensor_pos, enemies, ships):
        self.detected_enemies = []
        current_time = time.time()
        
        # Scan for enemies
        for enemy in enemies:
            distance = math.sqrt((enemy['pos'][0] - sensor_pos[0])**2 + 
                               (enemy['pos'][1] - sensor_pos[1])**2)
            
            if distance <= self.scan_radius:
                dx = enemy['velocity'][0]
                dy = enemy['velocity'][1]
                speed = math.sqrt(dx**2 + dy**2)
                direction = math.degrees(math.atan2(dy, dx)) % 360
                
                detection_data = {
                    'position': enemy['pos'],
                    'speed': round(speed, 2),
                    'direction': round(direction, 1),
                    'distance': round(distance, 2),
                    'threat_level': 'HIGH' if distance < 3 else 'MEDIUM',
                    'timestamp': current_time,
                    'id': enemy['id'],
                    'type': 'ENEMY'
                }
                
                self.detected_enemies.append(detection_data)
                
                log_entry = f"T:{int(current_time)%1000}|ENEMY#{enemy['id']}|Pos({enemy['pos'][0]:.1f},{enemy['pos'][1]:.1f})|Spd:{speed:.2f}"
                if log_entry not in self.detection_log[-50:]:
                    self.detection_log.append(log_entry)
        
        # Scan for ships
        for ship in ships:
            distance = math.sqrt((ship['pos'][0] - sensor_pos[0])**2 + 
                               (ship['pos'][1] - sensor_pos[1])**2)
            
            if distance <= self.scan_radius:
                dx = ship['velocity'][0]
                dy = ship['velocity'][1]
                speed = math.sqrt(dx**2 + dy**2)
                
                detection_data = {
                    'position': ship['pos'],
                    'speed': round(speed, 2),
                    'distance': round(distance, 2),
                    'threat_level': 'SHIP',
                    'id': ship['id'],
                    'type': 'SHIP'
                }
                
                self.detected_enemies.append(detection_data)
                
                log_entry = f"T:{int(current_time)%1000}|SHIP#{ship['id']}|Pos({ship['pos'][0]:.1f},{ship['pos'][1]:.1f})|Spd:{speed:.2f}"
                if log_entry not in self.detection_log[-50:]:
                    self.detection_log.append(log_entry)
        
        return self.detected_enemies
    
    def get_sensor_log(self):
        return self.detection_log[-10:]

class DynamicEnemy:
    def __init__(self, enemy_id, start_pos):
        self.id = enemy_id
        self.pos = list(start_pos)
        self.velocity = [0, 0]  # Stationary - no movement
        self.path_history = []
        
    def update(self):
        # Enemies are now stationary - no movement
        pass
    
    def get_data(self):
        return {'id': self.id, 'pos': self.pos, 'velocity': self.velocity}

class MovingShip:
    def __init__(self, ship_id, start_pos):
        self.id = ship_id
        self.pos = list(start_pos)
        self.velocity = [random.uniform(-0.15, 0.15), random.uniform(-0.15, 0.15)]
        self.path_history = []
        
    def update(self):
        self.pos[0] += self.velocity[0]
        self.pos[1] += self.velocity[1]
        
        if self.pos[0] <= 0 or self.pos[0] >= GRID - 1:
            self.velocity[0] *= -1
            self.pos[0] = max(0, min(GRID - 1, self.pos[0]))
        
        if self.pos[1] <= 0 or self.pos[1] >= GRID - 1:
            self.velocity[1] *= -1
            self.pos[1] = max(0, min(GRID - 1, self.pos[1]))
        
        self.path_history.append(tuple(self.pos))
        if len(self.path_history) > 30:
            self.path_history.pop(0)
    
    def get_data(self):
        return {'id': self.id, 'pos': self.pos, 'velocity': self.velocity}

def draw_tree(surface, x, y, size):
    # Tree trunk
    trunk_width = size // 5
    trunk_height = size // 3
    pygame.draw.rect(surface, (101, 67, 33), 
                    (x + size//2 - trunk_width//2, y + 2*size//3, trunk_width, trunk_height))
    
    # Tree foliage (3 circles)
    pygame.draw.circle(surface, (34, 139, 34), (x + size//2, y + size//3), size//3)
    pygame.draw.circle(surface, (46, 125, 50), (x + size//3, y + size//2), size//4)
    pygame.draw.circle(surface, (56, 142, 60), (x + 2*size//3, y + size//2), size//4)

def draw_enemy(surface, x, y, size):
    # Enemy character (skull-like)
    # Head
    pygame.draw.circle(surface, (255, 0, 0), (x + size//2, y + size//2), size//3)
    # Eyes
    eye_size = size // 10
    pygame.draw.circle(surface, (0, 0, 0), (x + size//2 - size//6, y + size//2 - size//10), eye_size)
    pygame.draw.circle(surface, (0, 0, 0), (x + size//2 + size//6, y + size//2 - size//10), eye_size)
    # Angry eyebrows
    pygame.draw.line(surface, (0, 0, 0), 
                    (x + size//2 - size//4, y + size//3),
                    (x + size//2 - size//8, y + size//2 - size//6), 3)
    pygame.draw.line(surface, (0, 0, 0), 
                    (x + size//2 + size//4, y + size//3),
                    (x + size//2 + size//8, y + size//2 - size//6), 3)

def draw_ship(surface, x, y, size):
    # Ship body
    points = [
        (x + size//4, y + 2*size//3),
        (x + 3*size//4, y + 2*size//3),
        (x + 2*size//3, y + size - 5),
        (x + size//3, y + size - 5)
    ]
    pygame.draw.polygon(surface, (100, 100, 100), points)
    
    # Ship cabin
    pygame.draw.rect(surface, (150, 150, 150), (x + size//3, y + size//3, size//3, size//3))
    
    # Mast
    pygame.draw.line(surface, (80, 80, 80), (x + size//2, y + size//3), (x + size//2, y + 5), 3)
    
    # Flag
    flag_points = [(x + size//2, y + 5), (x + size//2 + size//4, y + size//6), (x + size//2, y + size//4)]
    pygame.draw.polygon(surface, (200, 0, 0), flag_points)

def draw_ocean_obstacle(surface, x, y, size, obs_type):
    if obs_type == 'ISLAND':
        pygame.draw.ellipse(surface, (210, 180, 140), (x + 5, y + 5, size - 10, size - 10))
        pygame.draw.ellipse(surface, (34, 139, 34), (x + 10, y + 10, size - 20, size - 20))
    elif obs_type == 'REEF':
        for i in range(3):
            offset = i * (size // 4)
            pygame.draw.circle(surface, (255, 127, 80), (x + offset + 10, y + size//2), size//6)
    elif obs_type == 'ROCK':
        points = [(x + size//2, y + 5), (x + size - 5, y + size//2), 
                  (x + 2*size//3, y + size - 5), (x + size//3, y + size - 5), (x + 5, y + size//2)]
        pygame.draw.polygon(surface, (120, 120, 120), points)
    elif obs_type == 'MINE':
        pygame.draw.circle(surface, (255, 165, 0), (x + size//2, y + size//2), size//3)
        for angle in [0, 90, 180, 270]:
            rad = math.radians(angle)
            sx = x + size//2 + int(size//3 * math.cos(rad))
            sy = y + size//2 + int(size//3 * math.sin(rad))
            ex = x + size//2 + int(size//2 * math.cos(rad))
            ey = y + size//2 + int(size//2 * math.sin(rad))
            pygame.draw.line(surface, (255, 140, 0), (sx, sy), (ex, ey), 3)
    elif obs_type == 'TREE':
        draw_tree(surface, x, y, size)

OBSTACLE_TYPES = {
    'ISLAND': {'color': (34, 139, 34), 'key': 'I'},
    'REEF': {'color': (255, 127, 80), 'key': 'F'},
    'ROCK': {'color': (120, 120, 120), 'key': 'O'},
    'MINE': {'color': (255, 165, 0), 'key': 'M'},
    'TREE': {'color': (46, 125, 50), 'key': 'T'}
}

static_obstacles = {
    'ISLAND': [(3, 4), (3, 5), (15, 3), (15, 4)],
    'REEF': [(8, 7), (8, 8), (12, 13)],
    'ROCK': [(5, 10), (14, 8)],
    'MINE': [(10, 5), (6, 15)],
    'TREE': [(7, 3), (11, 11), (16, 6)]
}

selected_obstacle_type = 'ISLAND'
ACTIONS = [(-1, 0), (1, 0), (0, -1), (0, 1), (-1, -1), (-1, 1), (1, -1), (1, 1)]

def get_all_static_obstacles():
    result = []
    for obs_list in static_obstacles.values():
        result.extend(obs_list)
    return result

def is_obstacle_at(pos, dynamic_enemies, moving_ships):
    if pos in get_all_static_obstacles():
        return True
    for enemy in dynamic_enemies:
        enemy_pos = enemy.get_data()['pos']
        distance = math.sqrt((enemy_pos[0] - pos[0])**2 + (enemy_pos[1] - pos[1])**2)
        if distance < 1.5:
            return True
    for ship in moving_ships:
        ship_pos = ship.get_data()['pos']
        distance = math.sqrt((ship_pos[0] - pos[0])**2 + (ship_pos[1] - pos[1])**2)
        if distance < 1.5:
            return True
    return False

def heuristic(a, b):
    return math.sqrt((a[0] - b[0])**2 + (a[1] - b[1])**2)

def a_star(start, goal, dynamic_enemies, moving_ships):
    open_set = [(0, start, [start])]
    closed_set = set()
    g_score = {start: 0}
    
    while open_set:
        _, current, path = heapq.heappop(open_set)
        
        if current == goal:
            return path, len(closed_set)
        
        if current in closed_set:
            continue
        
        closed_set.add(current)
        
        for dx, dy in ACTIONS:
            neighbor = (current[0] + dx, current[1] + dy)
            
            if not (0 <= neighbor[0] < GRID and 0 <= neighbor[1] < GRID):
                continue
            
            if is_obstacle_at(neighbor, dynamic_enemies, moving_ships):
                continue
            
            move_cost = 1.414 if abs(dx) + abs(dy) == 2 else 1.0
            tentative_g = g_score[current] + move_cost
            
            if neighbor in g_score and tentative_g >= g_score[neighbor]:
                continue
            
            g_score[neighbor] = tentative_g
            f_score = tentative_g + heuristic(neighbor, goal)
            heapq.heappush(open_set, (f_score, neighbor, path + [neighbor]))
    
    return [start], 0

def dijkstra(start, goal, dynamic_enemies, moving_ships):
    pq = [(0, start, [start])]
    visited = set()
    nodes_explored = 0
    
    while pq:
        cost, current, path = heapq.heappop(pq)
        
        if current == goal:
            return path, nodes_explored
        
        if current in visited:
            continue
        
        visited.add(current)
        nodes_explored += 1
        
        for dx, dy in ACTIONS:
            neighbor = (current[0] + dx, current[1] + dy)
            
            if not (0 <= neighbor[0] < GRID and 0 <= neighbor[1] < GRID):
                continue
            
            if is_obstacle_at(neighbor, dynamic_enemies, moving_ships):
                continue
            
            move_cost = 1.414 if abs(dx) + abs(dy) == 2 else 1.0
            new_cost = cost + move_cost
            heapq.heappush(pq, (new_cost, neighbor, path + [neighbor]))
    
    return [start], nodes_explored

def bfs(start, goal, dynamic_enemies, moving_ships):
    queue = deque([(start, [start])])
    visited = {start}
    nodes_explored = 0
    
    while queue:
        current, path = queue.popleft()
        nodes_explored += 1
        
        if current == goal:
            return path, nodes_explored
        
        for dx, dy in ACTIONS:
            neighbor = (current[0] + dx, current[1] + dy)
            
            if not (0 <= neighbor[0] < GRID and 0 <= neighbor[1] < GRID):
                continue
            
            if neighbor in visited or is_obstacle_at(neighbor, dynamic_enemies, moving_ships):
                continue
            
            visited.add(neighbor)
            queue.append((neighbor, path + [neighbor]))
    
    return [start], nodes_explored

ALGORITHMS = {
    'A*': {'func': a_star, 'key': '1', 'color': (0, 255, 100), 'desc': 'Optimal with heuristic'},
    'Dijkstra': {'func': dijkstra, 'key': '2', 'color': (100, 150, 255), 'desc': 'Guaranteed shortest'},
    'BFS': {'func': bfs, 'key': '3', 'color': (255, 200, 0), 'desc': 'Level exploration'}
}

def draw_ocean_grid():
    for i in range(GRID):
        for j in range(GRID):
            shade = 30 + int(10 * math.sin((i + j) * 0.5))
            color = (0, shade, shade + 80)
            pygame.draw.rect(screen, color, (j * CELL, i * CELL, CELL, CELL))
            pygame.draw.rect(screen, (0, 50, 100), (j * CELL, i * CELL, CELL, CELL), 1)

def draw_obstacles():
    for obs_type, positions in static_obstacles.items():
        for pos in positions:
            draw_ocean_obstacle(screen, pos[1] * CELL, pos[0] * CELL, CELL, obs_type)

def draw_dynamic_enemies(enemies):
    for enemy in enemies:
        data = enemy.get_data()
        pos = data['pos']
        
        # No trail for stationary enemies
        
        x, y = int(pos[1] * CELL), int(pos[0] * CELL)
        draw_enemy(screen, x, y, CELL)
        
        id_text = tiny_font.render(f"E{data['id']}", True, (255, 255, 255))
        screen.blit(id_text, (x + CELL//2 - 10, y + CELL - 15))

def draw_moving_ships(ships):
    for ship in ships:
        data = ship.get_data()
        pos = data['pos']
        
        if len(ship.path_history) > 1:
            for i in range(len(ship.path_history) - 1):
                start = (int(ship.path_history[i][1] * CELL + CELL//2), 
                        int(ship.path_history[i][0] * CELL + CELL//2))
                end = (int(ship.path_history[i+1][1] * CELL + CELL//2), 
                      int(ship.path_history[i+1][0] * CELL + CELL//2))
                pygame.draw.line(screen, (50, 50, 100), start, end, 2)
        
        x, y = int(pos[1] * CELL), int(pos[0] * CELL)
        draw_ship(screen, x, y, CELL)
        
        id_text = tiny_font.render(f"S{data['id']}", True, (255, 255, 255))
        screen.blit(id_text, (x + CELL//2 - 8, y + CELL - 15))

def draw_radar_scan(sensor_pos, sensor, radar_angle):
    x = int(sensor_pos[1] * CELL + CELL//2)
    y = int(sensor_pos[0] * CELL + CELL//2)
    radius = int(sensor.scan_radius * CELL)
    
    pygame.draw.circle(screen, (0, 255, 255), (x, y), radius, 2)
    
    angle = radar_angle
    end_x = x + int(radius * math.cos(math.radians(angle)))
    end_y = y + int(radius * math.sin(math.radians(angle)))
    pygame.draw.line(screen, (0, 200, 200), (x, y), (end_x, end_y), 2)

def draw_ui_panel(selected_algo, sensor1, sensor2, algorithm_stats, selected_obstacle_type):
    panel_x = GRID * CELL + 20
    panel_y = 20
    
    title = font.render("RADAR PATH PLANNING", True, (255, 215, 0))
    screen.blit(title, (panel_x, panel_y))
    panel_y += 35
    
    subtitle = small_font.render("Dual Robot Navigation", True, (200, 200, 200))
    screen.blit(subtitle, (panel_x, panel_y))
    panel_y += 50
    
    if selected_algo:
        algo_name = small_font.render(f"Active: {selected_algo}", True, (0, 255, 0))
        screen.blit(algo_name, (panel_x, panel_y))
        
        if selected_algo in algorithm_stats:
            stats = algorithm_stats[selected_algo]
            stats_text = tiny_font.render(f"Nodes: {stats['nodes']} | Path: {stats['length']}", True, (180, 180, 180))
            screen.blit(stats_text, (panel_x, panel_y + 25))
    panel_y += 65
    
    pygame.draw.line(screen, (100, 100, 100), (panel_x, panel_y), (panel_x + 450, panel_y), 2)
    panel_y += 10
    
    # Obstacle types segregation
    obs_title = small_font.render("OBSTACLES:", True, (200, 200, 200))
    screen.blit(obs_title, (panel_x, panel_y))
    panel_y += 30
    
    for obs_type, data in OBSTACLE_TYPES.items():
        count = len(static_obstacles[obs_type])
        color = data['color'] if selected_obstacle_type == obs_type else (100, 100, 100)
        
        # Draw colored square
        pygame.draw.rect(screen, data['color'], (panel_x, panel_y, 15, 15))
        pygame.draw.rect(screen, (200, 200, 200), (panel_x, panel_y, 15, 15), 1)
        
        text = tiny_font.render(f"{obs_type}: {count}", True, color)
        screen.blit(text, (panel_x + 20, panel_y))
        panel_y += 22
    
    panel_y += 10
    pygame.draw.line(screen, (100, 100, 100), (panel_x, panel_y), (panel_x + 450, panel_y), 2)
    panel_y += 10
    
    algo_title = small_font.render("ALGORITHMS:", True, (200, 200, 200))
    screen.blit(algo_title, (panel_x, panel_y))
    panel_y += 30
    
    for name, data in ALGORITHMS.items():
        color = data['color'] if selected_algo == name else (100, 100, 100)
        text = small_font.render(f"{data['key']}: {name}", True, color)
        screen.blit(text, (panel_x, panel_y))
        desc = tiny_font.render(data['desc'], True, (150, 150, 150))
        screen.blit(desc, (panel_x + 15, panel_y + 20))
        panel_y += 45
    
    panel_y += 10
    pygame.draw.line(screen, (100, 100, 100), (panel_x, panel_y), (panel_x + 450, panel_y), 2)
    panel_y += 10
    
    # Robot 1 Sensor
    sensor_title = small_font.render("ROBOT 1 SENSOR", True, (0, 255, 255))
    screen.blit(sensor_title, (panel_x, panel_y))
    panel_y += 25
    
    detected1 = sensor1.detected_enemies
    if detected1:
        for item in detected1[:2]:
            item_type = "ENEMY" if item['type'] == 'ENEMY' else "SHIP"
            item_text = tiny_font.render(
                f"{item_type}#{item['id']}: {item['threat_level']}", True, (255, 100, 100))
            screen.blit(item_text, (panel_x, panel_y))
            panel_y += 18
    else:
        no_detect = tiny_font.render("No threats", True, (100, 255, 100))
        screen.blit(no_detect, (panel_x, panel_y))
        panel_y += 18
    
    panel_y += 10
    
    # Robot 2 Sensor
    sensor_title = small_font.render("ROBOT 2 SENSOR", True, (255, 255, 100))
    screen.blit(sensor_title, (panel_x, panel_y))
    panel_y += 25
    
    detected2 = sensor2.detected_enemies
    if detected2:
        for item in detected2[:2]:
            item_type = "ENEMY" if item['type'] == 'ENEMY' else "SHIP"
            item_text = tiny_font.render(
                f"{item_type}#{item['id']}: {item['threat_level']}", True, (255, 100, 100))
            screen.blit(item_text, (panel_x, panel_y))
            panel_y += 18
    else:
        no_detect = tiny_font.render("No threats", True, (100, 255, 100))
        screen.blit(no_detect, (panel_x, panel_y))
        panel_y += 18
    
    panel_y += 15
    pygame.draw.line(screen, (100, 100, 100), (panel_x, panel_y), (panel_x + 450, panel_y), 2)
    panel_y += 10
    
    controls_title = small_font.render("CONTROLS:", True, (200, 200, 200))
    screen.blit(controls_title, (panel_x, panel_y))
    panel_y += 30
    
    controls = [
        ("1", "A* Algorithm"),
        ("2", "Dijkstra Algorithm"),
        ("3", "BFS Algorithm"),
        ("SPACE", "Calculate & Run Path"),
        ("P", "Pause / Resume"),
        ("R", "Reset Simulation"),
        ("C", "Clear All Obstacles"),
        ("E", "Add Enemy (Stationary)"),
        ("H", "Add Moving Ship"),
        ("I", "Select Island Obstacle"),
        ("F", "Select Reef Obstacle"),
        ("O", "Select Rock Obstacle"),
        ("M", "Select Mine Obstacle"),
        ("T", "Select Tree Obstacle"),
        ("Left Click", "Place Obstacle"),
        ("Right Click", "Remove Obstacle"),
        ("ESC", "Exit Program")
    ]
    
    for key, desc in controls:
        key_text = tiny_font.render(f"{key}:", True, (100, 200, 255))
        desc_text = tiny_font.render(desc, True, (180, 180, 180))
        screen.blit
