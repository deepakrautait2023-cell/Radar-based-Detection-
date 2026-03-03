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
        screen.blit(key_text, (panel_x, panel_y))
        screen.blit(desc_text, (panel_x + 90, panel_y))
        panel_y += 18

# Main simulation variables
selected_algo = None
robot1_path = []
robot2_path = []
robot1_pos = START_STATE_1
robot2_pos = START_STATE_2
step_index = 0
simulation_running = False
path_calculated = False
is_paused = False
radar_angle = 0
frame_counter = 0

iot_sensor_1 = IoTSensor()
iot_sensor_2 = IoTSensor()

dynamic_enemies = [
    DynamicEnemy(1, [10.0, 10.0]),
    DynamicEnemy(2, [15.0, 8.0]),
    DynamicEnemy(3, [7.0, 14.0])
]

moving_ships = [
    MovingShip(1, [12.0, 5.0]),
    MovingShip(2, [6.0, 12.0])
]

algorithm_stats = {}
running = True

while running:
    clock.tick(15)
    
    # Always update enemies and ships (continuous movement)
    # Enemies are now stationary - no update needed
    
    for ship in moving_ships:
        ship.update()
    
    # Always scan area (continuous scanning)
    iot_sensor_1.scan_area(robot1_pos, [e.get_data() for e in dynamic_enemies], [s.get_data() for s in moving_ships])
    iot_sensor_2.scan_area(robot2_pos, [e.get_data() for e in dynamic_enemies], [s.get_data() for s in moving_ships])
    
    # Update radar angle
    radar_angle = (radar_angle + 12) % 360
    
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        
        if event.type == pygame.KEYDOWN:
            if event.key == pygame.K_ESCAPE:
                running = False
            
            # Algorithm selection
            for name, data in ALGORITHMS.items():
                if event.key == getattr(pygame, f'K_{data["key"]}'):
                    selected_algo = name
                    path_calculated = False
                    simulation_running = False
            
            # Obstacle type selection
            if event.key == pygame.K_i:
                selected_obstacle_type = 'ISLAND'
            if event.key == pygame.K_f:
                selected_obstacle_type = 'REEF'
            if event.key == pygame.K_o:
                selected_obstacle_type = 'ROCK'
            if event.key == pygame.K_m:
                selected_obstacle_type = 'MINE'
            if event.key == pygame.K_t:
                selected_obstacle_type = 'TREE'
            
            # Add enemy
            if event.key == pygame.K_e:
                new_id = max([e.id for e in dynamic_enemies], default=0) + 1
                new_pos = [random.uniform(5, GRID-5), random.uniform(5, GRID-5)]
                dynamic_enemies.append(DynamicEnemy(new_id, new_pos))
            
            # Add ship
            if event.key == pygame.K_h:
                new_id = max([s.id for s in moving_ships], default=0) + 1
                new_pos = [random.uniform(5, GRID-5), random.uniform(5, GRID-5)]
                moving_ships.append(MovingShip(new_id, new_pos))
            
            # Calculate path and start simulation
            if event.key == pygame.K_SPACE and selected_algo:
                algo_func = ALGORITHMS[selected_algo]['func']
                robot1_path, nodes1 = algo_func(START_STATE_1, GOAL_STATE, dynamic_enemies, moving_ships)
                robot2_path, nodes2 = algo_func(START_STATE_2, GOAL_STATE, dynamic_enemies, moving_ships)
                
                algorithm_stats[selected_algo] = {
                    'nodes': (nodes1 + nodes2) // 2,
                    'length': (len(robot1_path) + len(robot2_path)) // 2
                }
                
                robot1_pos = START_STATE_1
                robot2_pos = START_STATE_2
                step_index = 0
                simulation_running = True
                path_calculated = True
                is_paused = False
                frame_counter = 0
            
            # Pause/Resume
            if event.key == pygame.K_p and path_calculated:
                is_paused = not is_paused
            
            # Reset
            if event.key == pygame.K_r:
                robot1_path = []
                robot2_path = []
                robot1_pos = START_STATE_1
                robot2_pos = START_STATE_2
                step_index = 0
                simulation_running = False
                path_calculated = False
                is_paused = False
                algorithm_stats = {}
                frame_counter = 0
            
            # Clear obstacles
            if event.key == pygame.K_c:
                for key in static_obstacles:
                    static_obstacles[key] = []
        
        if event.type == pygame.MOUSEBUTTONDOWN:
            mx, my = pygame.mouse.get_pos()
            grid_x = my // CELL
            grid_y = mx // CELL
            
            if 0 <= grid_x < GRID and 0 <= grid_y < GRID:
                pos = (grid_x, grid_y)
                
                if event.button == 1:  # Left click - add obstacle
                    if pos != START_STATE_1 and pos != START_STATE_2 and pos != GOAL_STATE:
                        if pos not in get_all_static_obstacles():
                            static_obstacles[selected_obstacle_type].append(pos)
                
                elif event.button == 3:  # Right click - remove obstacle
                    for obs_type in static_obstacles:
                        if pos in static_obstacles[obs_type]:
                            static_obstacles[obs_type].remove(pos)
    
    # Move robots along path (SLOWER - every 3 frames)
    if simulation_running and not is_paused:
        frame_counter += 1
        if frame_counter >= 3:  # Move every 3 frames (slower speed)
            frame_counter = 0
            if robot1_path and step_index < len(robot1_path):
                robot1_pos = robot1_path[step_index]
            if robot2_path and step_index < len(robot2_path):
                robot2_pos = robot2_path[step_index]
            step_index += 1
            
            if step_index >= max(len(robot1_path), len(robot2_path)):
                simulation_running = False
    
    # Drawing
    screen.fill((0, 20, 40))
    draw_ocean_grid()
    draw_obstacles()
    draw_dynamic_enemies(dynamic_enemies)
    draw_moving_ships(moving_ships)
    
    # Draw START markers
    pygame.draw.rect(screen, (255, 255, 0), 
                    (START_STATE_1[1] * CELL + 3, START_STATE_1[0] * CELL + 3, CELL - 6, CELL - 6), 3)
    pygame.draw.rect(screen, (255, 255, 100), 
                    (START_STATE_2[1] * CELL + 3, START_STATE_2[0] * CELL + 3, CELL - 6, CELL - 6), 3)
    
    # Draw GOAL marker
    pygame.draw.rect(screen, (0, 255, 0), 
                    (GOAL_STATE[1] * CELL + 5, GOAL_STATE[0] * CELL + 5, CELL - 10, CELL - 10), 4)
    goal_text = tiny_font.render("GOAL", True, (0, 255, 0))
    screen.blit(goal_text, (GOAL_STATE[1] * CELL + 10, GOAL_STATE[0] * CELL + 10))
    
    # Draw paths
    if robot1_path and selected_algo:
        color = ALGORITHMS[selected_algo]['color']
        for i in range(len(robot1_path) - 1):
            start = (robot1_path[i][1] * CELL + CELL//2, robot1_path[i][0] * CELL + CELL//2)
            end = (robot1_path[i+1][1] * CELL + CELL//2, robot1_path[i+1][0] * CELL + CELL//2)
            pygame.draw.line(screen, color, start, end, 4)
    
    if robot2_path and selected_algo:
        color_dim = tuple(c // 2 for c in ALGORITHMS[selected_algo]['color'])
        for i in range(len(robot2_path) - 1):
            start = (robot2_path[i][1] * CELL + CELL//2, robot2_path[i][0] * CELL + CELL//2)
            end = (robot2_path[i+1][1] * CELL + CELL//2, robot2_path[i+1][0] * CELL + CELL//2)
            pygame.draw.line(screen, color_dim, start, end, 3)
    
    # Draw continuous radar scans
    draw_radar_scan(robot1_pos, iot_sensor_1, radar_angle)
    draw_radar_scan(robot2_pos, iot_sensor_2, radar_angle + 180)
    
    # Draw robots
    robot1_x = robot1_pos[1] * CELL + CELL//2
    robot1_y = robot1_pos[0] * CELL + CELL//2
    pygame.draw.circle(screen, (0, 255, 255), (robot1_x, robot1_y), CELL//3)
    pygame.draw.circle(screen, (0, 200, 200), (robot1_x, robot1_y), CELL//3, 3)
    robot1_text = tiny_font.render("R1", True, (0, 0, 0))
    screen.blit(robot1_text, (robot1_x - 10, robot1_y - 5))
    
    robot2_x = robot2_pos[1] * CELL + CELL//2
    robot2_y = robot2_pos[0] * CELL + CELL//2
    pygame.draw.circle(screen, (255, 255, 100), (robot2_x, robot2_y), CELL//3)
    pygame.draw.circle(screen, (200, 200, 0), (robot2_x, robot2_y), CELL//3, 3)
    robot2_text = tiny_font.render("R2", True, (0, 0, 0))
    screen.blit(robot2_text, (robot2_x - 10, robot2_y - 5))
    
    # Draw UI panel
    draw_ui_panel(selected_algo, iot_sensor_1, iot_sensor_2, algorithm_stats, selected_obstacle_type)
    
    # Draw status
    status_y = HEIGHT - 40
    if is_paused:
        status = small_font.render("PAUSED - Press P to resume", True, (255, 255, 0))
        screen.blit(status, (20, status_y))
    elif simulation_running:
        status = small_font.render("RUNNING - Press P to pause", True, (0, 255, 0))
        screen.blit(status, (20, status_y))
    elif path_calculated:
        status = small_font.render("READY - Path calculated", True, (100, 200, 255))
        screen.blit(status, (20, status_y))
    else:
        status = small_font.render("STATIC - Select algorithm and press SPACE", True, (200, 200, 200))
        screen.blit(status, (20, status_y))
    
    pygame.display.flip()

pygame.quit()
sys.exit()
