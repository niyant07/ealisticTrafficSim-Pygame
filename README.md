# realisticTrafficSim-Pygame
A Pygame-based traffic simulation featuring a winding road with realistic car behaviors like lane-based overtaking, dynamic checkpoint navigation for a player car, and bidirectional AI traffic. Watch red player cars dodge blue traffic while heading to checkpoints—perfect for visualizing road dynamics and simple AI pathfinding!
[car.py](https://github.com/user-attachments/files/25310153/car.py)
import pygame
import math
import random

pygame.init()

# ---------------- CONFIG ----------------
W, H = 1000, 600
ROAD_WIDTH = 90
BASE_LANE_OFFSET = 22
OVERTAKE_OFFSET = 18
FPS = 60

WHITE = (245, 245, 245)
BLACK = (15, 15, 15)
GRAY = (120, 120, 120)
RED = (220, 50, 50)
BLUE = (60, 120, 220)
GREEN = (40, 180, 40)
BROWN = (120, 70, 20)
YELLOW = (255, 210, 0)

screen = pygame.display.set_mode((W, H))
pygame.display.set_caption("Realistic Traffic Behaviour")
clock = pygame.time.Clock()
font = pygame.font.SysFont(None, 20)

# ---------------- ROAD ----------------
road = [(50, 320), (260, 320), (420, 200), (620, 200), (760, 420), (950, 420)]

# ---------------- ROAD DRAW ----------------
def draw_road(points, width):
    left, right = [], []
    for i in range(len(points)-1):
        x1, y1 = points[i]
        x2, y2 = points[i+1]
        dx, dy = x2-x1, y2-y1
        l = math.hypot(dx, dy)
        nx, ny = -dy/l, dx/l
        left.append((x1 + nx*width/2, y1 + ny*width/2))
        right.append((x1 - nx*width/2, y1 - ny*width/2))
    x, y = points[-1]
    left.append((x + nx*width/2, y + ny*width/2))
    right.append((x - nx*width/2, y - ny*width/2))
    pygame.draw.polygon(screen, BLACK, left + right[::-1])
    pygame.draw.lines(screen, GRAY, False, points, 2)

# ---------------- PROJECTION ----------------
def project(seg, t, lane, offset=0):
    x1, y1 = road[seg]
    x2, y2 = road[seg+1]
    px = x1 + (x2-x1)*t
    py = y1 + (y2-y1)*t
    dx, dy = x2-x1, y2-y1
    l = math.hypot(dx, dy)
    nx, ny = -dy/l, dx/l
    return px + nx*(BASE_LANE_OFFSET*lane + offset), py + ny*(BASE_LANE_OFFSET*lane + offset)

# ---------------- CHECKPOINT ----------------
class Checkpoint:
    def __init__(self, seg, is_home=False):
        self.seg = seg
        self.t = 0.5
        self.is_home = is_home
        self.x, self.y = project(seg, self.t, 1)

    def draw(self):
        pygame.draw.circle(screen, GREEN if self.is_home else YELLOW,
                           (int(self.x), int(self.y)), 10)
        screen.blit(font.render("Home" if self.is_home else "CP", True, BLACK),
                    (self.x-14, self.y-26))

    def clicked(self, mx, my):
        return math.hypot(mx-self.x, my-self.y) < 14

# ---------------- TREE ----------------
class Tree:
    def __init__(self):
        while True:
            self.x = random.randint(0, W)
            self.y = random.randint(0, H)
            if min(math.hypot(self.x-x, self.y-y) for x,y in road) > ROAD_WIDTH:
                break

    def draw(self):
        pygame.draw.circle(screen, GREEN, (self.x, self.y), 12)
        pygame.draw.rect(screen, BROWN, (self.x-3, self.y, 6, 15))

# ---------------- CAR ----------------
class Car:
    def __init__(self, lane, color):
        self.lane = lane
        self.seg = 0
        self.t = 0.0
        self.speed = 0.008
        self.angle = 0
        self.target = None
        self.overtake_offset = 0.0
        self.x, self.y = project(self.seg, self.t, self.lane)

        self.image = pygame.Surface((26,12), pygame.SRCALPHA)
        self.image.fill(color)

    def go_to_checkpoint(self, cp):
        self.target = (cp.seg, cp.t)

    def update(self, cars):
        # movement
        if self.target:
            ts, tt = self.target
            if self.seg == ts and abs(self.t-tt) < 0.02:
                self.t = tt
                self.target = None
            else:
                self.t += self.speed
                if self.t >= 1:
                    self.t = 0
                    self.seg = min(self.seg+1, len(road)-2)

        # reset offset
        self.overtake_offset *= 0.85

        for c in cars:
            if c == self:
                continue

            # SAME direction only
            if c.lane == self.lane:
                same_seg = self.seg == c.seg
                close = abs(self.t - c.t) < 0.12

                if same_seg and close:
                    # rear car overtakes
                    if self.t < c.t:
                        self.overtake_offset = -self.lane * OVERTAKE_OFFSET
                    break

        self.x, self.y = project(self.seg, self.t, self.lane, self.overtake_offset)
        x1, y1 = road[self.seg]
        x2, y2 = road[self.seg+1]
        self.angle = math.degrees(math.atan2(y2-y1, x2-x1))

    def draw(self):
        rot = pygame.transform.rotate(self.image, -self.angle)
        screen.blit(rot, rot.get_rect(center=(self.x, self.y)))

# ---------------- TRAFFIC ----------------
class TrafficCar(Car):
    def __init__(self, forward=True):
        super().__init__(1 if forward else -1, BLUE)
        self.seg = random.randint(0, len(road)-2)
        self.speed = random.uniform(0.004, 0.007)

    def update(self, cars):
        self.t += self.speed
        if self.t >= 1:
            self.t = 0
            self.seg = (self.seg+1) % (len(road)-1)
        super().update(cars)

# ---------------- INIT ----------------
player = Car(1, RED)

checkpoints = [
    Checkpoint(1),
    Checkpoint(2),
    Checkpoint(3),
    Checkpoint(4, is_home=True)
]

traffic = [TrafficCar(True) for _ in range(4)] + [TrafficCar(False) for _ in range(4)]
trees = [Tree() for _ in range(30)]

# ---------------- LOOP ----------------
running = True
while running:
    screen.fill(WHITE)

    for e in pygame.event.get():
        if e.type == pygame.QUIT:
            running = False
        if e.type == pygame.MOUSEBUTTONDOWN:
            mx, my = pygame.mouse.get_pos()
            for cp in checkpoints:
                if cp.clicked(mx, my):
                    player.go_to_checkpoint(cp)

    draw_road(road, ROAD_WIDTH)

    for t in trees:
        t.draw()
    for cp in checkpoints:
        cp.draw()

    cars = [player] + traffic
    player.update(cars)
    for t in traffic:
        t.update(cars)

    for t in traffic:
        t.draw()
    player.draw()

    pygame.display.flip()
    clock.tick(FPS)

pygame.quit()
