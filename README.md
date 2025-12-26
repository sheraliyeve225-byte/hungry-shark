import pygame
import random
import math
import sys
from enum import Enum

# PyGame ni ishga tushirish
pygame.init()
pygame.font.init()

# O'yin sozlamalari
WIDTH, HEIGHT = 1000, 700
FPS = 60
TILE_SIZE = 50
MAP_SIZE = 20  # 20x20 tiles

# Ranglar
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED = (255, 50, 50)
GREEN = (0, 180, 0)
BROWN = (139, 69, 19)
GRAY = (128, 128, 128)
BLUE = (50, 150, 255)
YELLOW = (255, 255, 0)
DARK_GREEN = (0, 100, 0)
TAN = (210, 180, 140)

# O'yin oynasini yaratish
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("PUBG Mobile - Lite Version")
clock = pygame.time.Clock()

# Qurol turlari
class WeaponType(Enum):
    ASSAULT_RIFLE = 1
    SHOTGUN = 2
    SNIPER = 3

# Qurol klassi
class Weapon:
    def __init__(self, weapon_type):
        self.type = weapon_type
        self.damage = 0
        self.range = 0
        self.fire_rate = 0
        self.ammo = 0
        self.max_ammo = 0
        self.reload_time = 0
        self.name = ""
        self.color = WHITE
        
        if weapon_type == WeaponType.ASSAULT_RIFLE:
            self.damage = 25
            self.range = 300
            self.fire_rate = 10
            self.ammo = 30
            self.max_ammo = 30
            self.reload_time = 2.5
            self.name = "M416"
            self.color = GRAY
        elif weapon_type == WeaponType.SHOTGUN:
            self.damage = 50
            self.range = 100
            self.fire_rate = 1
            self.ammo = 8
            self.max_ammo = 8
            self.reload_time = 3.0
            self.name = "S686"
            self.color = BROWN
        elif weapon_type == WeaponType.SNIPER:
            self.damage = 80
            self.range = 500
            self.fire_rate = 1.5
            self.ammo = 5
            self.max_ammo = 5
            self.reload_time = 4.0
            self.name = "Kar98k"
            self.color = DARK_GREEN
    
    def shoot(self):
        if self.ammo > 0:
            self.ammo -= 1
            return True
        return False
    
    def reload(self):
        self.ammo = self.max_ammo

# O'yinchi klassi
class Player:
    def __init__(self, x, y, is_player=True):
        self.x = x
        self.y = y
        self.width = 30
        self.height = 50
        self.speed = 5
        self.health = 100
        self.max_health = 100
        self.is_player = is_player
        self.weapon = Weapon(WeaponType.ASSAULT_RIFLE)
        self.direction = 0  # 0: right, 180: left
        self.score = 0
        self.reload_timer = 0
        self.inventory = {
            "bandage": 3,
            "medkit": 1,
            "energy_drink": 2
        }
        
    def draw(self, camera_x, camera_y):
        # O'yinchi tanasi
        player_rect = pygame.Rect(
            self.x - camera_x, 
            self.y - camera_y, 
            self.width, 
            self.height
        )
        
        # Tanani chizish
        color = BLUE if self.is_player else RED
        pygame.draw.rect(screen, color, player_rect)
        
        # Qurolni chizish
        weapon_length = 30
        angle_rad = math.radians(self.direction)
        end_x = self.x - camera_x + math.cos(angle_rad) * weapon_length
        end_y = self.y - camera_y + math.sin(angle_rad) * weapon_length
        
        pygame.draw.line(
            screen, 
            self.weapon.color, 
            (self.x - camera_x, self.y - camera_y),
            (end_x, end_y),
            5
        )
        
        # Sog'liq chizig'i
        health_width = self.width
        health_height = 5
        health_ratio = self.health / self.max_health
        
        # Sog'liq chizig'i fon
        pygame.draw.rect(
            screen, 
            RED,
            (
                self.x - camera_x,
                self.y - camera_y - 10,
                health_width,
                health_height
            )
        )
        
        # Sog'liq chizig'i
        pygame.draw.rect(
            screen, 
            GREEN,
            (
                self.x - camera_x,
                self.y - camera_y - 10,
                health_width * health_ratio,
                health_height
            )
        )
    
    def move(self, dx, dy, obstacles):
        new_x = self.x + dx * self.speed
        new_y = self.y + dy * self.speed
        
        # To'siqlarni tekshirish
        player_rect = pygame.Rect(new_x, new_y, self.width, self.height)
        can_move = True
        
        for obstacle in obstacles:
            if player_rect.colliderect(obstacle.get_rect()):
                can_move = False
                break
        
        if can_move:
            self.x = new_x
            self.y = new_y
    
    def get_rect(self):
        return pygame.Rect(self.x, self.y, self.width, self.height)
    
    def take_damage(self, damage):
        self.health -= damage
        if self.health < 0:
            self.health = 0
    
    def heal(self, amount):
        self.health += amount
        if self.health > self.max_health:
            self.health = self.max_health

# Dushman klassi (AI)
class Enemy(Player):
    def __init__(self, x, y):
        super().__init__(x, y, False)
        self.weapon = random.choice([
            Weapon(WeaponType.ASSAULT_RIFLE),
            Weapon(WeaponType.SHOTGUN),
            Weapon(WeaponType.SNIPER)
        ])
        self.move_timer = 0
        self.shoot_timer = 0
        self.target_x = x
        self.target_y = y
    
    def update(self, player, obstacles, bullets):
        # Oddiy AI mantiqi
        self.move_timer += 1
        self.shoot_timer += 1
        
        # Agar o'yinchi yaqin bo'lsa, nishonga olish
        distance_to_player = math.sqrt((self.x - player.x)**2 + (self.y - player.y)**2)
        
        if distance_to_player < 400:  # Ko'rish masofasi
            # O'q otish
            if self.shoot_timer > 60 / self.weapon.fire_rate and distance_to_player < self.weapon.range:
                self.shoot_timer = 0
                if self.weapon.shoot():
                    # O'q yo'nalishi
                    angle = math.atan2(player.y - self.y, player.x - self.x)
                    self.direction = math.degrees(angle)
                    
                    # O'q yaratish
                    bullet = Bullet(
                        self.x, 
                        self.y, 
                        angle, 
                        self.weapon.damage, 
                        self.weapon.range,
                        self
                    )
                    bullets.append(bullet)
                elif self.weapon.ammo == 0:
                    self.weapon.reload()
            
            # Harakatlanish
            if self.move_timer > 30:
                self.move_timer = 0
                if distance_to_player > 150:  # O'q otish uchun optimal masofa
                    # O'yinchiga yaqinlashish
                    angle = math.atan2(player.y - self.y, player.x - self.x)
                    dx = math.cos(angle)
                    dy = math.sin(angle)
                    self.move(dx, dy, obstacles)
                    self.direction = math.degrees(angle)
                else:
                    # O'q otish uchun masofani saqlash
                    dx = random.uniform(-1, 1)
                    dy = random.uniform(-1, 1)
                    self.move(dx, dy, obstacles)
                    self.direction = math.degrees(math.atan2(dy, dx))
        else:
            # Tasodifiy harakat
            if self.move_timer > 60:
                self.move_timer = 0
                self.target_x = self.x + random.randint(-200, 200)
                self.target_y = self.y + random.randint(-200, 200)
            
            # Maqsadga harakat
            if self.target_x != self.x or self.target_y != self.y:
                angle = math.atan2(self.target_y - self.y, self.target_x - self.x)
                dx = math.cos(angle) * 0.5
                dy = math.sin(angle) * 0.5
                self.move(dx, dy, obstacles)
                self.direction = math.degrees(angle)

# O'q klassi
class Bullet:
    def __init__(self, x, y, angle, damage, max_range, shooter):
        self.x = x
        self.y = y
        self.angle = angle
        self.speed = 15
        self.damage = damage
        self.max_range = max_range
        self.distance_traveled = 0
        self.shooter = shooter
        
    def update(self):
        self.x += math.cos(self.angle) * self.speed
        self.y += math.sin(self.angle) * self.speed
        self.distance_traveled += self.speed
        
    def draw(self, camera_x, camera_y):
        pygame.draw.circle(
            screen, 
            YELLOW, 
            (int(self.x - camera_x), int(self.y - camera_y)), 
            3
        )
    
    def is_out_of_range(self):
        return self.distance_traveled > self.max_range

# To'siq klassi
class Obstacle:
    def __init__(self, x, y, width, height, obstacle_type="tree"):
        self.x = x
        self.y = y
        self.width = width
        self.height = height
        self.type = obstacle_type
        
    def draw(self, camera_x, camera_y):
        rect = pygame.Rect(
            self.x - camera_x,
            self.y - camera_y,
            self.width,
            self.height
        )
        
        if self.type == "tree":
            pygame.draw.rect(screen, BROWN, rect)
            # Daraxt toji
            pygame.draw.circle(
                screen, 
                GREEN, 
                (int(self.x + self.width//2 - camera_x), int(self.y - camera_y)), 
                25
            )
        elif self.type == "rock":
            pygame.draw.ellipse(screen, GRAY, rect)
        elif self.type == "house":
            pygame.draw.rect(screen, TAN, rect)
            # Tom
            points = [
                (self.x - camera_x, self.y - camera_y),
                (self.x + self.width//2 - camera_x, self.y - 30 - camera_y),
                (self.x + self.width - camera_x, self.y - camera_y)
            ]
            pygame.draw.polygon(screen, RED, points)
    
    def get_rect(self):
        return pygame.Rect(self.x, self.y, self.width, self.height)

# Loot (o'lja) klassi
class Loot:
    def __init__(self, x, y, loot_type):
        self.x = x
        self.y = y
        self.type = loot_type
        self.width = 20
        self.height = 20
        
        if loot_type == "weapon":
            self.color = GRAY
            self.name = random.choice(["M416", "S686", "Kar98k"])
        elif loot_type == "medkit":
            self.color = RED
            self.name = "Medkit"
        elif loot_type == "bandage":
            self.color = WHITE
            self.name = "Bandage"
        elif loot_type == "ammo":
            self.color = YELLOW
            self.name = "Ammo"
    
    def draw(self, camera_x, camera_y):
        rect = pygame.Rect(
            self.x - camera_x,
            self.y - camera_y,
            self.width,
            self.height
        )
        pygame.draw.rect(screen, self.color, rect)
        
        # Ichki chiziq
        inner_rect = pygame.Rect(
            self.x + 2 - camera_x,
            self.y + 2 - camera_y,
            self.width - 4,
            self.height - 4
        )
        pygame.draw.rect(screen, BLACK, inner_rect, 1)
        
        # Nom
        font = pygame.font.SysFont(None, 12)
        text = font.render(self.name, True, WHITE)
        screen.blit(text, (self.x - camera_x, self.y + self.height - camera_y))
    
    def get_rect(self):
        return pygame.Rect(self.x, self.y, self.width, self.height)

# Xarita yaratish
def create_map():
    obstacles = []
    loot_items = []
    
    # Daraxtlar
    for _ in range(20):
        x = random.randint(0, MAP_SIZE * TILE_SIZE)
        y = random.randint(0, MAP_SIZE * TILE_SIZE)
        obstacles.append(Obstacle(x, y, 20, 40, "tree"))
    
    # Toshlar
    for _ in range(15):
        x = random.randint(0, MAP_SIZE * TILE_SIZE)
        y = random.randint(0, MAP_SIZE * TILE_SIZE)
        obstacles.append(Obstacle(x, y, 30, 20, "rock"))
    
    # Uylar
    for _ in range(3):
        x = random.randint(0, MAP_SIZE * TILE_SIZE)
        y = random.randint(0, MAP_SIZE * TILE_SIZE)
        obstacles.append(Obstacle(x, y, 80, 60, "house"))
    
    # Loot elementlari
    for _ in range(20):
        x = random.randint(0, MAP_SIZE * TILE_SIZE)
        y = random.randint(0, MAP_SIZE * TILE_SIZE)
        loot_type = random.choice(["weapon", "medkit", "bandage", "ammo"])
        loot_items.append(Loot(x, y, loot_type))
    
    return obstacles, loot_items

# O'yin interfeysi
class GameUI:
    def __init__(self):
        self.font = pygame.font.SysFont(None, 24)
        self.big_font = pygame.font.SysFont(None, 48)
    
    def draw(self, player, enemies, game_state):
        # O'yinchi ma'lumotlari
        info_y = 10
        
        # Sog'liq
        health_text = self.font.render(f"Sog'lik: {player.health}/100", True, GREEN)
        screen.blit(health_text, (10, info_y))
        
        # Qurol ma'lumoti
        weapon_text = self.font.render(
            f"Qurol: {player.weapon.name} ({player.weapon.ammo}/{player.weapon.max_ammo})", 
            True, 
            WHITE
        )
        screen.blit(weapon_text, (10, info_y + 30))
        
        # Ball
        score_text = self.font.render(f"Ball: {player.score}", True, YELLOW)
        screen.blit(score_text, (10, info_y + 60))
        
        # Qolgan dushmanlar
        enemies_text = self.font.render(f"Dushmanlar: {len(enemies)}", True, RED)
        screen.blit(enemies_text, (10, info_y + 90))
        
        # Inventory
        inv_text = self.font.render("Inventory:", True, WHITE)
        screen.blit(inv_text, (WIDTH - 150, info_y))
        
        inv_items = [
            f"Bandaj: {player.inventory['bandage']}",
            f"Medkit: {player.inventory['medkit']}",
            f"Ichimlik: {player.inventory['energy_drink']}"
        ]
        
        for i, item in enumerate(inv_items):
            item_text = self.font.render(item, True, WHITE)
            screen.blit(item_text, (WIDTH - 150, info_y + 30 + i * 30))
        
        # Boshqaruv ko'rsatmalari
        controls = [
            "Boshqaruv:",
            "WASD - Harakat",
            "Maus - Nishonga olish",
            "Chap Klik - Otish",
            "R - Qurolni to'ldirish",
            "1 - Bandaj ishlatish",
            "2 - Medkit ishlatish",
            "3 - Ichimlik ishlatish"
        ]
        
        for i, control in enumerate(controls):
            control_text = self.font.render(control, True, WHITE)
            screen.blit(control_text, (WIDTH - 300, HEIGHT - 200 + i * 25))
        
        # O'yin holati
        if game_state == "game_over":
            game_over_text = self.big_font.render("OYIN TUGADI!", True, RED)
            restart_text = self.font.render("ENTER - Qayta boshlash, ESC - Chiqish", True, WHITE)
            
            screen.blit(game_over_text, (WIDTH//2 - 100, HEIGHT//2 - 50))
            screen.blit(restart_text, (WIDTH//2 - 150, HEIGHT//2 + 20))
        
        elif game_state == "victory":
            victory_text = self.big_font.render("G'ALABA!", True, GREEN)
            restart_text = self.font.render("ENTER - Qayta boshlash, ESC - Chiqish", True, WHITE)
            
            screen.blit(victory_text, (WIDTH//2 - 80, HEIGHT//2 - 50))
            screen.blit(restart_text, (WIDTH//2 - 150, HEIGHT//2 + 20))

# Asosiy o'yin funksiyasi
def main():
    # O'yin obyektlarini yaratish
    player = Player(WIDTH//2, HEIGHT//2)
    enemies = [Enemy(random.randint(100, 900), random.randint(100, 500)) for _ in range(5)]
    obstacles, loot_items = create_map()
    bullets = []
    ui = GameUI()
    
    camera_x = player.x - WIDTH//2
    camera_y = player.y - HEIGHT//2
    
    game_state = "playing"  # playing, game_over, victory
    mouse_x, mouse_y = 0, 0
    
    # O'yin asosiy tsikli
    running = True
    while running:
        # Kamerani yangilash
        camera_x = player.x - WIDTH//2
        camera_y = player.y - HEIGHT//2
        
        # Hodisalarni tekshirish
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            
            elif event.type == pygame.MOUSEMOTION:
                mouse_x, mouse_y = event.pos
                # Sichqoncha yo'nalishi bo'yicha o'yinchi yo'nalishini o'zgartirish
                rel_x = mouse_x + camera_x - player.x
                rel_y = mouse_y + camera_y - player.y
                player.direction = math.degrees(math.atan2(rel_y, rel_x))
            
            elif event.type == pygame.MOUSEBUTTONDOWN:
                if event.button == 1 and game_state == "playing":  # Chap zichqoncha tugmasi
                    if player.weapon.shoot():
                        # O'q yaratish
                        angle = math.radians(player.direction)
                        bullet = Bullet(
                            player.x, 
                            player.y, 
                            angle, 
                            player.weapon.damage, 
                            player.weapon.range,
                            player
                        )
                        bullets.append(bullet)
                    else:
                        player.weapon.reload()
            
            elif event.type == pygame.KEYDOWN:
                if game_state == "playing":
                    if event.key == pygame.K_r:
                        player.weapon.reload()
                    
                    # Inventory elementlarini ishlatish
                    elif event.key == pygame.K_1:
                        if player.inventory["bandage"] > 0:
                            player.heal(25)
                            player.inventory["bandage"] -= 1
                    
                    elif event.key == pygame.K_2:
                        if player.inventory["medkit"] > 0:
                            player.heal(75)
                            player.inventory["medkit"] -= 1
                    
                    elif event.key == pygame.K_3:
                        if player.inventory["energy_drink"] > 0:
                            player.speed += 2
                            player.inventory["energy_drink"] -= 1
                            # 10 soniyadan keyin tezlik normal holatga qaytadi
                            pygame.time.set_timer(pygame.USEREVENT, 10000, True)
                
                elif game_state in ["game_over", "victory"]:
                    if event.key == pygame.K_RETURN:
                        # O'yinni qayta boshlash
                        player = Player(WIDTH//2, HEIGHT//2)
                        enemies = [Enemy(random.randint(100, 900), random.randint(100, 500)) for _ in range(5)]
                        obstacles, loot_items = create_map()
                        bullets = []
                        game_state = "playing"
                    
                    elif event.key == pygame.K_ESCAPE:
                        running = False
        
        # O'yin holati "playing" bo'lsa
        if game_state == "playing":
            # O'yinchi harakatini boshqarish
            keys = pygame.key.get_pressed()
            dx, dy = 0, 0
            
            if keys[pygame.K_w] or keys[pygame.K_UP]:
                dy -= 1
            if keys[pygame.K_s] or keys[pygame.K_DOWN]:
                dy += 1
            if keys[pygame.K_a] or keys[pygame.K_LEFT]:
                dx -= 1
            if keys[pygame.K_d] or keys[pygame.K_RIGHT]:
                dx += 1
            
            # Normalizatsiya
            if dx != 0 or dy != 0:
                length = math.sqrt(dx*dx + dy*dy)
                dx /= length
                dy /= length
            
            player.move(dx, dy, obstacles)
            
            # Dushmanlarni yangilash
            for enemy in enemies[:]:
