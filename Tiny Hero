import time
import random
import board
from digitalio import DigitalInOut, Direction
from PIL import Image, ImageDraw, ImageOps, ImageFont
from adafruit_rgb_display import st7789

# Display 설정
cs_pin = DigitalInOut(board.CE0)
dc_pin = DigitalInOut(board.D25)
reset_pin = DigitalInOut(board.D24)
BAUDRATE = 100000000

spi = board.SPI()
disp = st7789.ST7789(
    spi,
    height=240,
    y_offset=80,
    rotation=180,
    cs=cs_pin,
    dc=dc_pin,
    rst=reset_pin,
    baudrate=BAUDRATE,
)

# 조이스틱 버튼 설정
button_U = DigitalInOut(board.D17)  # Up
button_D = DigitalInOut(board.D22)  # Down
button_L = DigitalInOut(board.D27)  # Left
button_R = DigitalInOut(board.D23)  # Right
button_attack = DigitalInOut(board.D5)  # Attack 버튼

for button in [button_U, button_D, button_L, button_R, button_attack]:
    button.direction = Direction.INPUT

# 스프라이트 시트 및 배경 타일 로드
player_sprite = Image.open("/home/han/Desktop/embeded/player.png").convert("RGBA")
slime_sprite = Image.open("/home/han/Desktop/embeded/slime.png").convert("RGBA")
skeleton_sprite = Image.open("/home/han/Desktop/embeded/skeleton.png").convert("RGBA") 
background_tile = Image.open("/home/han/Desktop/embeded/hills3.png").convert("RGB")
health_potion_sprite = Image.open("/home/han/Desktop/embeded/potion3.png").convert("RGBA")  

# 스프라이트 크기 설정
player_width, player_height = 48, 48
slime_width, slime_height = 32, 32
skeleton_width, skeleton_height = 48, 48  
attack_effect_width, attack_effect_height = 64, 64  # 공격 효과 이미지 크기 설정
health_potion_width, health_potion_height = 16, 16  # 체력 포션 크기 설정
scale_factor = 2
frame_duration = 0.1

# 확대된 캐릭터 및 슬라임, 스켈레톤 크기 계산
scaled_player_width = player_width * scale_factor
scaled_player_height = player_height * scale_factor
scaled_slime_width = slime_width * scale_factor
scaled_slime_height = slime_height * scale_factor
scaled_skeleton_width = skeleton_width * scale_factor
scaled_skeleton_height = skeleton_height * scale_factor
scaled_attack_effect_width = attack_effect_width * scale_factor
scaled_attack_effect_height = attack_effect_height * scale_factor
scaled_health_potion_width = health_potion_width * scale_factor
scaled_health_potion_height = health_potion_height * scale_factor

# 맵 크기 설정
map_width = disp.width * 4
map_height = disp.height * 4

# 배경 맵 생성
background = Image.new("RGB", (map_width, map_height))
tile_width, tile_height = background_tile.size
for i in range(0, map_width, tile_width):
    for j in range(0, map_height, tile_height):
        background.paste(background_tile, (i, j))

# 초기 위치 설정
x = map_width // 2
y = map_height // 2
move_step = 10
slime_move_step = 2  
skeleton_move_step = 1.5  
offset_x = 0
offset_y = 0  # 화면 오프셋

# 주인공 체력 및 점수 설정
player_hp = 10  
max_hp = 10
player_score = 0
attack_distance = 50  

# 에너지 설정
player_stamina = 10
max_stamina = 10
stamina_bar_width = 60
stamina_bar_height = 8
stamina_recovery_interval = 1.0  # 에너지 회복 간격 (초)
last_stamina_recovery_time = time.monotonic()

# 무적 시간 설정
invincible = False
invincibility_duration = 2.0  # 무적 시간 (초)
last_hit_time = 0

# 체력 바 설정
health_bar_width = 60
health_bar_height = 8

# 폰트 설정 (점수판을 위해)
font = ImageFont.load_default()

# 주인공 애니메이션 프레임 맵핑
frame_map = {
    "down": [(3, i) for i in range(3)],
    "up": [(5, i) for i in range(3)],
    "left": [(4, i) for i in range(3)],
    "right": [(4, i) for i in range(3)],
    "attack_down": [(6, i) for i in range(3)],
    "attack_up": [(8, i) for i in range(3)],
    "attack_side": [(7, i) for i in range(3)],  # 좌우 공격 애니메이션 통합
    "idle": [(0, i) for i in range(3)]
}

# 슬라임 애니메이션 프레임 맵핑
slime_map = {
    "idle": [(0, i) for i in range(3)],
    "attack": [(1, i) for i in range(3)],
    "death": [(12, i) for i in range(2)]
}

# 스켈레톤 애니메이션 프레임 맵핑 
skeleton_map = {
    "idle": [(0, i) for i in range(6)],      # 첫 번째 행
    "move": [(1, i) for i in range(6)],      # 두 번째 행
    "attack": [(2, i) for i in range(6)],    # 세 번째 행
    "death": [(3, i) for i in range(6)],     # 네 번째 행
    "hit": [(4, i) for i in range(4)]        # 다섯 번째 행: 공격받는 모션
}

# 슬라임과 스켈레톤 상태를 저장하는 리스트
slimes = []
skeletons = []
last_slime_spawn_time = time.monotonic()
last_skeleton_spawn_time = time.monotonic()
last_frame_time = time.monotonic()
frame_index = 0
action_state = 'idle' 
current_direction = "down"
SLIME_SPAWN_INTERVAL = 1.0  # 슬라임 생성 주기
SKELETON_SPAWN_INTERVAL = 5.0  # 스켈레톤 생성 주기

# 체력 포션 리스트
health_potions = []

# Game Over 상태 추가
game_over = False  # 게임 오버 상태를 추적하는 변수 추가

# 슬라임 생성 함수
def spawn_slime():
    """플레이어와 일정 거리 이상 떨어진 가장자리에서 슬라임을 생성."""
    min_distance = 200  # 플레이어와 최소 거리 설정
    max_attempts = 10  # 최대 시도 횟수
    for _ in range(max_attempts):
        edge = random.choice(["top", "bottom", "left", "right"])
        if edge == "top":
            slime_pos = [random.randint(0, map_width - scaled_slime_width), 0]
        elif edge == "bottom":
            slime_pos = [random.randint(0, map_width - scaled_slime_width), map_height - scaled_slime_height]
        elif edge == "left":
            slime_pos = [0, random.randint(0, map_height - scaled_slime_height)]
        else:
            slime_pos = [map_width - scaled_slime_width, random.randint(0, map_height - scaled_slime_height)]

        # 플레이어와의 거리 계산
        distance = ((slime_pos[0] - x) ** 2 + (slime_pos[1] - y) ** 2) ** 0.5
        if distance >= min_distance:
            slimes.append({
                "pos": slime_pos,
                "direction": random.choice(["up", "down", "left", "right"]),
                "frame_index": 0,
                "alive": True,
                "dying": False,
                "hp": 1,
                "attacking": False
            })
            break

# 스켈레톤 생성 함수
def spawn_skeleton():
    """플레이어와 일정 거리 이상 떨어진 가장자리에서 스켈레톤을 생성."""
    min_distance = 200  # 플레이어와 최소 거리 설정
    max_attempts = 10  # 최대 시도 횟수
    for _ in range(max_attempts):
        edge = random.choice(["top", "bottom", "left", "right"])
        if edge == "top":
            skeleton_pos = [random.randint(0, map_width - scaled_skeleton_width), 0]
        elif edge == "bottom":
            skeleton_pos = [random.randint(0, map_width - scaled_skeleton_width), map_height - scaled_skeleton_height]
        elif edge == "left":
            skeleton_pos = [0, random.randint(0, map_height - scaled_skeleton_height)]
        else:
            skeleton_pos = [map_width - scaled_skeleton_width, random.randint(0, map_height - scaled_skeleton_height)]

        # 플레이어와의 거리 계산
        distance = ((skeleton_pos[0] - x) ** 2 + (skeleton_pos[1] - y) ** 2) ** 0.5
        if distance >= min_distance:
            skeletons.append({
                "pos": skeleton_pos,
                "direction": random.choice(["up", "down", "left", "right"]),
                "frame_index": 0,
                "alive": True,
                "dying": False,
                "hp": 3,  # 스켈레톤 체력을 3으로 설정
                "attacking": False,
                "hit": False  # 초기값 추가
            })
            break

# 스프라이트 크기 조정 함수
def get_scaled_sprite(sheet, row, col, width, height, flip=False):
    box = (col * width, row * height, (col + 1) * width, (row + 1) * height)
    sprite = sheet.crop(box).resize((width * scale_factor, height * scale_factor), Image.NEAREST).convert("RGBA")
    if flip:
        sprite = ImageOps.mirror(sprite)
    return sprite

# 메인 게임 루프
while True:
    flip = False
    attack_flip = False
    moving = False
    current_time = time.monotonic()

        # 게임 오버 상태 체크
    if game_over:
        game_over_screen = Image.new("RGB", (disp.width, disp.height), "black")
        draw = ImageDraw.Draw(game_over_screen)
    
        # "Game Over" 텍스트
        game_over_text = "Game Over"
        try:
            font = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 32)
        except IOError:
            font = ImageFont.load_default()
        text_width, text_height = draw.textsize(game_over_text, font=font)
        x_text = (disp.width - text_width) // 2
        y_text = (disp.height - text_height) // 3 
        draw.text((x_text, y_text), game_over_text, font=font, fill="white")
    
        # 플레이어 점수 텍스트
        score_text = f"Score: {player_score}"
        score_font = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 24)
        score_text_width, score_text_height = draw.textsize(score_text, font=score_font)
        x_score = (disp.width - score_text_width) // 2
        y_score = y_text + text_height + 20  
        draw.text((x_score, y_score), score_text, font=score_font, fill="white")
    
        disp.image(game_over_screen)
        break  # 게임 루프 종료


    # 슬라임 생성 주기 조정
    if current_time - last_slime_spawn_time > SLIME_SPAWN_INTERVAL:
        spawn_slime()
        last_slime_spawn_time = current_time

    # 스켈레톤 생성 주기 조정
    if (
        player_score >= 300 and  # 스코어가 300점 이상일 때
        len(skeletons) < 5 and   # 스켈레톤의 수가 5마리 미만일 때
        current_time - last_skeleton_spawn_time > SKELETON_SPAWN_INTERVAL
    ):
        spawn_skeleton()
        last_skeleton_spawn_time = current_time

    # 에너지 회복
    if current_time - last_stamina_recovery_time > stamina_recovery_interval:
        if player_stamina < max_stamina:
            player_stamina += 1
        last_stamina_recovery_time = current_time

    # 무적 시간 해제
    if invincible and current_time - last_hit_time > invincibility_duration:
        invincible = False

    # 애니메이션 상태 관리
    if action_state == 'attacking':
        # 공격 애니메이션 중에는 다른 입력을 처리하지 않음
        pass
    else:
        # 공격 버튼 확인
        if not button_attack.value and player_stamina > 0:
            action_state = 'attacking'
            frame_index = 0
            player_stamina -= 1
            attack_direction = current_direction  # 공격 방향 저장
            if attack_direction == "left":
                attack_flip = True
            else:
                attack_flip = False

            # 슬라임 공격 범위 확인
            slime_hit_count = 0
            for slime in slimes:
                if slime_hit_count >= 3:
                    break
                if slime["alive"] and not slime["dying"]:
                    slime_x, slime_y = slime["pos"]
                    # 공격 범위 내에 있는지 확인
                    distance = ((slime_x - x) ** 2 + (slime_y - y) ** 2) ** 0.5
                    if distance <= attack_distance:
                        slime["hp"] -= 1
                        if slime["hp"] <= 0:
                            slime["dying"] = True
                            slime["frame_index"] = 0
                            player_score += 10
                            slime_hit_count += 1
                            # 포션 드랍 확률 조정 (20%로 설정)
                            if random.randint(1, 100) <= 20:
                                health_potions.append({"pos": [slime_x, slime_y], "collected": False})

            # 스켈레톤 공격 범위 확인
            skeleton_hit_count = 0
            for skeleton in skeletons:
                if skeleton_hit_count >= 3:
                    break
                if skeleton["alive"] and not skeleton["dying"]:
                    skeleton_x, skeleton_y = skeleton["pos"]
                    # 공격 범위 내에 있는지 확인
                    distance = ((skeleton_x - x) ** 2 + (skeleton_y - y) ** 2) ** 0.5
                    if distance <= attack_distance:
                        skeleton["hp"] -= 1
                        if skeleton["hp"] <= 0:
                            skeleton["dying"] = True
                            skeleton["frame_index"] = 0
                            player_score += 50  # 스켈레톤 처치 시 더 많은 점수
                            skeleton_hit_count += 1
                            # 포션 드랍 확률 조정 (50%로 설정)
                            if random.randint(1, 100) <= 50:
                                health_potions.append({"pos": [skeleton_x, skeleton_y], "collected": False})
                        else:
                            skeleton["hit"] = True  # 공격받는 상태로 변경
                            skeleton["frame_index"] = 0
        else:
            # 조이스틱 이동 처리
            if not button_U.value:
                y = max(0, y - move_step)
                current_direction = "up"
                flip = False
                moving = True
                action_state = 'moving'
            elif not button_D.value:
                y = min(map_height - scaled_player_height, y + move_step)
                current_direction = "down"
                flip = False
                moving = True
                action_state = 'moving'
            elif not button_L.value:
                x = max(0, x - move_step)
                current_direction = "left"
                flip = True
                moving = True
                action_state = 'moving'
            elif not button_R.value:
                x = min(map_width - scaled_player_width, x + move_step)
                current_direction = "right"
                flip = False
                moving = True
                action_state = 'moving'
            else:
                # 움직이지 않을 때
                action_state = 'idle'

    # 애니메이션 프레임 업데이트
    if current_time - last_frame_time > frame_duration:
        last_frame_time = current_time
        if action_state == 'attacking':
            frame_index += 1
            if current_direction in ['left', 'right']:
                anim_key = 'attack_side'
            else:
                anim_key = 'attack_' + current_direction
            if frame_index >= len(frame_map[anim_key]):
                action_state = 'idle'
                frame_index = 0
        elif action_state == 'moving':
            frame_index = (frame_index + 1) % len(frame_map[current_direction])
        else:
            frame_index = (frame_index + 1) % len(frame_map['idle'])

    # 슬라임과 플레이어 충돌 체크
    if not invincible:
        player_center_x = x + scaled_player_width / 2
        player_center_y = y + scaled_player_height / 2
        for slime in slimes:
            if slime["alive"] and not slime["dying"]:
                slime_x, slime_y = slime["pos"]
                slime_center_x = slime_x + scaled_slime_width / 2
                slime_center_y = slime_y + scaled_slime_height / 2
                # 중심점 거리 계산
                distance = ((player_center_x - slime_center_x) ** 2 + (player_center_y - slime_center_y) ** 2) ** 0.5
                if distance < 30:  # 슬라임과의 충돌 거리 설정
                    player_hp = max(player_hp - 1, 0)  
                    if player_hp <= 0:
                        game_over = True  # 게임 오버 상태로 변경
                    invincible = True
                    last_hit_time = current_time
                    break
        # 스켈레톤과 플레이어 충돌 체크
        for skeleton in skeletons:
            if skeleton["alive"] and not skeleton["dying"]:
                skeleton_x, skeleton_y = skeleton["pos"]
                skeleton_center_x = skeleton_x + scaled_skeleton_width / 2
                skeleton_center_y = skeleton_y + scaled_skeleton_height / 2
                # 중심점 거리 계산
                distance = ((player_center_x - skeleton_center_x) ** 2 + (player_center_y - skeleton_center_y) ** 2) ** 0.5
                if distance < 30:  # 스켈레톤과의 충돌 거리 설정
                    player_hp = max(player_hp - 2, 0)  # 체력이 0까지 내려갈 수 있도록 수정
                    if player_hp <= 0:
                        game_over = True  # 게임 오버 상태로 변경
                    invincible = True
                    last_hit_time = current_time
                    break

    # 체력 포션과 플레이어 충돌 체크
    player_center_x = x + scaled_player_width / 2
    player_center_y = y + scaled_player_height / 2
    for potion in health_potions:
        if not potion["collected"]:
            potion_x, potion_y = potion["pos"]
            potion_center_x = potion_x + scaled_health_potion_width / 2
            potion_center_y = potion_y + scaled_health_potion_height / 2
            # 중심점 거리 계산
            distance = ((player_center_x - potion_center_x) ** 2 + (player_center_y - potion_center_y) ** 2) ** 0.5
            if distance < 20:  # 포션 획득 거리 설정
                potion["collected"] = True
                player_hp = min(player_hp + 1, max_hp)

    # 화면 오프셋 설정
    offset_x = min(max(x - disp.width // 2, 0), map_width - disp.width)
    offset_y = min(max(y - disp.height // 2, 0), map_height - disp.height)

    # 배경 및 주인공, 슬라임, 스켈레톤 그리기
    temp_background = background.crop((offset_x, offset_y, offset_x + disp.width, offset_y + disp.height))

    # 캐릭터 애니메이션 프레임 선택
    if action_state == 'attacking':
        if current_direction in ['left', 'right']:
            anim_key = 'attack_side'
            frame_coords = frame_map[anim_key][frame_index]
            character_frame = get_scaled_sprite(player_sprite, *frame_coords, player_width, player_height, attack_flip)
        else:
            anim_key = 'attack_' + current_direction
            frame_coords = frame_map[anim_key][frame_index]
            character_frame = get_scaled_sprite(player_sprite, *frame_coords, player_width, player_height)
    elif action_state == 'moving':
        frame_coords = frame_map[current_direction][frame_index]
        character_frame = get_scaled_sprite(player_sprite, *frame_coords, player_width, player_height, flip)
    else:
        frame_coords = frame_map['idle'][frame_index]
        character_frame = get_scaled_sprite(player_sprite, *frame_coords, player_width, player_height, flip)

    # 무적 상태 시 반투명 효과 적용
    if invincible and int(time.monotonic() * 10) % 2 == 0:
        for i in range(character_frame.size[0]):
            for j in range(character_frame.size[1]):
                r, g, b, a = character_frame.getpixel((i, j))
                if a > 0:
                    character_frame.putpixel((i, j), (r, g, b, 128))
    temp_background.paste(character_frame, (x - offset_x, y - offset_y), character_frame)

    # 슬라임 이동 및 그리기
    for slime in slimes:
        if slime["alive"] and not slime["dying"]:
            slime_x, slime_y = slime["pos"]
            flip_slime = False
            # 슬라임이 플레이어를 향해 부드럽게 이동
            dx = x - slime_x
            dy = y - slime_y
            distance = (dx ** 2 + dy ** 2) ** 0.5
            if distance != 0:
                slime_x += (dx / distance) * slime_move_step
                slime_y += (dy / distance) * slime_move_step
                slime_x = max(0, min(slime_x, map_width - scaled_slime_width))
                slime_y = max(0, min(slime_y, map_height - scaled_slime_height))
                slime["pos"] = [slime_x, slime_y]
            if dx < 0:
                flip_slime = True
            slime_frame = get_scaled_sprite(slime_sprite, *slime_map["idle"][int(frame_index) % len(slime_map["idle"])], slime_width, slime_height, flip_slime)
            temp_background.paste(slime_frame, (int(slime_x - offset_x), int(slime_y - offset_y)), slime_frame)
        elif slime["dying"]:
            if slime["frame_index"] < len(slime_map["death"]):
                slime_frame = get_scaled_sprite(slime_sprite, *slime_map["death"][slime["frame_index"]], slime_width, slime_height)
                temp_background.paste(slime_frame, (int(slime["pos"][0] - offset_x), int(slime["pos"][1] - offset_y)), slime_frame)
                slime["frame_index"] += 1
            else:
                slime["alive"] = False

    # 스켈레톤 이동 및 그리기
    for skeleton in skeletons:
        if skeleton["alive"]:
            if skeleton["hp"] <= 0:  # 스켈레톤이 죽을 때
                skeleton["dying"] = True
                skeleton["frame_index"] = 0
            elif skeleton.get("hit", False):  # 공격받는 상태
                if skeleton["frame_index"] < len(skeleton_map["hit"]):
                    skeleton_frame = get_scaled_sprite(
                        skeleton_sprite,
                        *skeleton_map["hit"][int(skeleton["frame_index"])],
                        skeleton_width, skeleton_height
                    )
                    temp_background.paste(skeleton_frame, (int(skeleton["pos"][0] - offset_x), int(skeleton["pos"][1] - offset_y)), skeleton_frame)
                    skeleton["frame_index"] += 0.2
                else:
                    skeleton["hit"] = False  # 공격받는 애니메이션 완료 후
                    skeleton["frame_index"] = 0
            else:
                # 기존 이동 애니메이션 처리
                skeleton_x, skeleton_y = skeleton["pos"]
                flip_skeleton = False
                # 스켈레톤이 플레이어를 향해 부드럽게 이동
                dx = x - skeleton_x
                dy = y - skeleton_y
                distance = (dx ** 2 + dy ** 2) ** 0.5
                if distance != 0:
                    skeleton_x += (dx / distance) * skeleton_move_step
                    skeleton_y += (dy / distance) * skeleton_move_step
                    skeleton_x = max(0, min(skeleton_x, map_width - scaled_skeleton_width))
                    skeleton_y = max(0, min(skeleton_y, map_height - scaled_skeleton_height))
                    skeleton["pos"] = [skeleton_x, skeleton_y]
                if dx < 0:
                    flip_skeleton = True
                skeleton_frame = get_scaled_sprite(skeleton_sprite, *skeleton_map["move"][int(skeleton["frame_index"]) % len(skeleton_map["move"])], skeleton_width, skeleton_height, flip_skeleton)
                temp_background.paste(skeleton_frame, (int(skeleton_x - offset_x), int(skeleton_y - offset_y)), skeleton_frame)
                skeleton["frame_index"] += 0.2  # 프레임 인덱스 업데이트
        elif skeleton["dying"]:
            if skeleton["frame_index"] < len(skeleton_map["death"]):
                skeleton_frame = get_scaled_sprite(skeleton_sprite, *skeleton_map["death"][int(skeleton["frame_index"])], skeleton_width, skeleton_height)
                temp_background.paste(skeleton_frame, (int(skeleton["pos"][0] - offset_x), int(skeleton["pos"][1] - offset_y)), skeleton_frame)
                skeleton["frame_index"] += 0.2
            else:
                skeleton["alive"] = False

    # 체력 포션 그리기
    for potion in health_potions:
        if not potion["collected"]:
            potion_x, potion_y = potion["pos"]
            potion_frame = health_potion_sprite.resize(
                (scaled_health_potion_width, scaled_health_potion_height), Image.NEAREST
            ).convert("RGBA")  
            temp_background.paste(potion_frame, (int(potion_x - offset_x), int(potion_y - offset_y)), potion_frame)

    # 점수 및 체력 바 그리기
    draw = ImageDraw.Draw(temp_background)
    health_ratio = player_hp / max_hp
    health_bar_length = int(health_bar_width * health_ratio)
    draw.rectangle((10, 10, 10 + health_bar_width, 10 + health_bar_height), outline="black", fill="gray")
    draw.rectangle((10, 10, 10 + health_bar_length, 10 + health_bar_height), outline="black", fill="red")

    # 에너지 바 그리기
    stamina_ratio = player_stamina / max_stamina
    stamina_bar_length = int(stamina_bar_width * stamina_ratio)
    draw.rectangle((10, 25, 10 + stamina_bar_width, 25 + stamina_bar_height), outline="black", fill="gray")
    draw.rectangle((10, 25, 10 + stamina_bar_length, 25 + stamina_bar_height), outline="black", fill="yellow")

    # 점수 그리기
    draw.text((10, 45), f"Score: {player_score}", font=font, fill="white")

    disp.image(temp_background)
    time.sleep(0.02)
