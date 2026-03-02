import pygame
import sys
import random
from time import sleep
import os

SCREEN_WIDTH = 480 #게임 화면의 가로크기
SCREEN_HEIGHT = 640 #게임 화면의 세로 크기


BLACK = (0,0,0)
WHITE = (200,200,200)
YELLOW = (250, 250, 50)
RED = (250,50,50)

#전역변수
FPS = 60

#드론 객체
class Fighter(pygame.sprite.Sprite):
    def __init__(self):
        super(Fighter, self).__init__()
        self.image = pygame.image.load(resource_path('assets/fighter.png'))
        self.rect = self.image.get_rect()
        self.reset()


    #드론 리셋
    def reset(self):
        self.rect.x = int(SCREEN_WIDTH / 2)
        self.rect.y = SCREEN_HEIGHT - self.rect.height
        self.dx = 0
        self.dy =0

    #드론 업데이트
    def update(self):
        self.rect.x += self.dx
        self.rect.y += self.dy

        if self.rect.x < 0 or self.rect.x + self.rect.width > SCREEN_WIDTH:
            self.rect.x -= self.dx

        if self.rect.y < 0 or self.rect.y + self.rect.height > SCREEN_HEIGHT:
            self.rect.y -= self.dy
    
    #드론 그리기
    def draw(self, screen):
        screen.blit(self.image, self.rect)
        

    #드론 충돌 체크
    def collide(self, sprites):
        for sprite in sprites:
            if pygame.sprite.collide_rect(self,sprite):
                return sprite


# 미사일 객체
class Missile(pygame.sprite.Sprite):
    def __init__(self, xpos, ypos, speed):
        super(Missile, self).__init__()
        self.image = pygame.image.load(resource_path('assets/missile.png'))
        self.sound = pygame.mixer.Sound(resource_path('assets/missile.wav'))
        self.rect = self.image.get_rect()
        self.rect.x =  xpos
        self.rect.y = ypos
        self.speed = speed

    
    # 미사일 발사
    def launch(self):
        self.sound.play()

    #미사일 업데이트
    def update(self):
        self.rect.y -= self.speed
        if self.rect.y + self.rect.height <0:
            self.kill()

    #미사일 충돌 체크
    def collide(self, sprites):
        for sprite in sprites:
            if pygame.sprite.collide_rect(self, sprite):
                return sprite

# 몬스터 개체
class Rock(pygame.sprite.Sprite):
    def __init__(self, xpos, ypos, speed):
        super(Rock, self).__init__()
        rock_images_path = resource_path('assets/rock')
        image_file_list = os.listdir(rock_images_path)
        self.image_path_list = [os.path.join(rock_images_path,file) for file in image_file_list if file.endswith(".png")]
        choice_rock_path = random.choice(self.image_path_list)
        self.image = pygame.image.load(choice_rock_path)
        self.rect = self.image.get_rect()
        self.rect.x = xpos
        self.rect.y = ypos
        self.speed = speed
        self.is_transformed = False  # 충돌 여부
        self.transform_time = 0      # 변환 시점

    def update(self):
        if not self.is_transformed:
            self.rect.y += self.speed
        elif pygame.time.get_ticks() - self.transform_time > 1500:  # 1.5초 후 삭제
            self.kill()

    def out_of_screen(self):
        if self.rect.y > SCREEN_HEIGHT:
            return True

    def transform(self):
        # 꽃이나 잎으로 변환
        flower_images = [resource_path('assets/flower.png'), resource_path('assets/grass.png')]
        self.image = pygame.image.load(random.choice(flower_images))
        self.is_transformed = True
        self.transform_time = pygame.time.get_ticks()



# 게임 객체
class Game():
    def __init__(self):
        self.menu_image = pygame.image.load(resource_path('assets/background.png'))
        self.background_image = pygame.image.load(resource_path('assets/background.png'))
        self.explosion_image = pygame.image.load(resource_path('assets/explosion.png'))
        self.default_font = pygame.font.Font(resource_path('assets/NanumGothic.ttf'),28)
        self.font_70 = pygame.font.Font(resource_path('assets/NanumGothic.ttf'),70)
        self.font_30 = pygame.font.Font(resource_path('assets/NanumGothic.ttf'),30)
        explosion_file = ('assets/explosion01.wav', 'assets/explosion02.wav', 'assets/explosion03.wav')
        self.explosion_path_list = [resource_path(file) for file in explosion_file]
        self.gameover_sound = pygame.mixer.Sound(resource_path('assets/gameover.wav'))
        pygame.mixer.music.load(resource_path('assets/music.wav'))
        self.menu_on = True
        self.game_over = False  # 게임 오버 상태 추가
        self.game_over_image = pygame.image.load(resource_path('assets/gameover.png'))
        self.background_image2 = pygame.image.load(resource_path('assets/background2.png'))  # 게임 오버 배경 추가
        
        self.fighter = Fighter()
        self.missiles = pygame.sprite.Group()
        self.rocks = pygame.sprite.Group()

        self.occur_prob = 40
        self.shot_count = 0
        self.count_missed =0

        #게임 메뉴 On/Off
        self.menu_on = True

    def process_events(self):
        #게임 이벤트 처리
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                return True
            
            # 메뉴 화면 이벤트 처리
            if self.menu_on:
                if event.type == pygame.KEYDOWN and event.key == pygame.K_SPACE:
                    pygame.mixer.music.play(-1)
                    self.shot_count = 0
                    self.count_missed = 0
                    self.menu_on = False

            # 게임 오버 또는 환경 정화 완료 이벤트 처리
            elif self.game_over or self.shot_count >= 100:  # 추가된 조건
                if event.type == pygame.KEYDOWN and event.key == pygame.K_SPACE:
                    pygame.mixer.music.play(-1)
                    self.shot_count = 0
                    self.count_missed = 0
                    self.rocks.empty()
                    self.fighter.reset()
                    self.game_over = False
                    self.menu_on = False

            # 게임 화면 이벤트 처리
            elif not self.menu_on:
                if event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_LEFT:
                        self.fighter.dx -= 5
                    elif event.key == pygame.K_RIGHT:
                        self.fighter.dx += 5
                    elif event.key == pygame.K_UP:
                        self.fighter.dy -= 5
                    elif event.key == pygame.K_DOWN:
                        self.fighter.dy += 5
                    elif event.key == pygame.K_SPACE:
                        missile = Missile(self.fighter.rect.centerx, self.fighter.rect.y, 10)
                        missile.launch()
                        self.missiles.add(missile)
                elif event.type == pygame.KEYUP:
                    if event.key == pygame.K_LEFT or event.key == pygame.K_RIGHT:
                        self.fighter.dx = 0
                    elif event.key == pygame.K_UP or event.key == pygame.K_DOWN:
                        self.fighter.dy = 0

        return False

    def run_logic(self, screen):
       if not self.menu_on and not self.game_over:  # 게임 진행 중일 때만 로직 처리
        #몬스터 생성 및 충돌 처리
        occur_of_rocks = 1 + int(self.shot_count / 300)
        min_rock_speed = 1 + int(self.shot_count / 200)
        max_rock_speed = 1 + int(self.shot_count / 100)

        if random.randint(1, self.occur_prob) == 1:
            for i in range(occur_of_rocks):
                speed = random.randint(min_rock_speed, max_rock_speed)
                rock = Rock(random.randint(0, SCREEN_WIDTH - 30), 0, speed)
                self.rocks.add(rock)

        # 미사일 충돌 체크
        for missile in self.missiles:
            rock = missile.collide(self.rocks)
            if rock and not rock.is_transformed:
                self.occur_explosion(screen, rock.rect.x, rock.rect.y)
                rock.transform()  #  꽃이나 잎으로 변환
                self.shot_count += 1
                missile.kill()

        # 몬스터 화면 벗어남 체크
        for rock in self.rocks:
            if not rock.is_transformed and rock.out_of_screen():
                rock.kill()
                self.count_missed += 1

        # 드론과 충돌하거나 몬스터를 3번 이상 놓쳤을 경우 게임 종료
        if self.fighter.collide(self.rocks) or self.count_missed >= 3:
            pygame.mixer_music.stop()
            self.occur_explosion(screen, self.fighter.rect.x, self.fighter.rect.y)
            self.gameover_sound.play()
            self.game_over = True  # 게임 오버 상태 전환
            sleep(1)

    #텍스트 그리기
    def draw_text(self, screen, text, font, x, y, color):
        text_obj = font.render(text, True, color)
        text_rect = text_obj.get_rect()
        text_rect.center = x, y
        screen.blit(text_obj, text_rect)

    #충돌 이벤트 발생
    def occur_explosion(self,screen, x, y):
        explosion_rect = self.explosion_image.get_rect()
        explosion_rect.x =x
        explosion_rect.y =y
        screen.blit(self.explosion_image, explosion_rect)
        pygame.display.update()

        explosion_sound = pygame.mixer.Sound(random.choice(self.explosion_path_list))
        explosion_sound.play()
    
    #게임 메뉴 출력
    def display_menu(self, screen):
        screen.blit(self.menu_image, [0,0])
        draw_x = int(SCREEN_WIDTH / 2)
        draw_y = int(SCREEN_HEIGHT / 4)
        self. draw_text(screen, '녹색 수호자', self.font_70, draw_x, draw_y, YELLOW)
        self.draw_text(screen, '스페이스 키를 누르면',self.font_30, draw_x,draw_y+200,WHITE)
        self.draw_text(screen, '게임이 시작 됩니다.',self.font_30,draw_x,draw_y+ 250, WHITE)

    #게임 프레임 출력
    def display_frame(self,screen):
        if not self.menu_on and not self.game_over and self.shot_count < 100:
        
            screen.blit(self.background_image, self.background_image.get_rect())
            self.draw_text(screen, f'제거된 몬스터: {self.shot_count}', self.default_font, 120, 40, YELLOW)
            self.draw_text(screen, f'놓친 몬스터: {self.count_missed}', self.default_font, SCREEN_WIDTH - 100, 40, RED)
            self.rocks.update()
            self.rocks.draw(screen)
            self.missiles.update()
            self.missiles.draw(screen)
            self.fighter.update()
            self.fighter.draw(screen)


    def display_game_over(self, screen):
    # 배경은 그대로 유지
        screen.blit(self.background_image, self.background_image.get_rect())
        # 게임 오버 이미지 화면 가운데 표시
        game_over_rect = self.game_over_image.get_rect(center=(SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2))
        screen.blit(self.game_over_image, game_over_rect)
        # 재시작 안내 텍스트
        self.draw_text(screen, '스페이스 키를 눌러 재시작', self.font_30, SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2 + 100, WHITE)

    def display_cleanup_complete(self, screen):
        # 배경을 변경
        screen.blit(self.background_image2, self.background_image2.get_rect())
        # 완료 텍스트 화면 가운데 표시
        self.draw_text(screen, '환경 정화 완료!', self.font_70, SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2 - 50, YELLOW)
        # 안내 텍스트 표시
        self.draw_text(screen, '스페이스 키를 눌러 재시작', self.font_30, SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2 + 50, WHITE)

def resource_path(relative_path):
    try:
        base_path = sys._MEIPASS
    except Exception:
        base_path = os.path.abspath(".")
    return os.path.join(base_path, relative_path)

def main():
    pygame.init()
    pygame.display.set_caption('Shooting Game')
    screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
    clock = pygame.time.Clock()
    game = Game()

    done = False
    while not done:
        done = game.process_events()

        if game.menu_on:  # 메뉴 화면 처리
            game.display_menu(screen)
        elif game.game_over:  # 게임 오버 화면 처리
            game.display_game_over(screen)
        elif game.shot_count >= 100:  # 환경 정화 완료 화면 처리
            game.display_cleanup_complete(screen)
        else:  # 게임 진행 화면 처리
            game.run_logic(screen)
            game.display_frame(screen)
            
        pygame.display.flip()
        clock.tick(FPS)

    pygame.quit()

if __name__ == "__main__":
    main()
    



                


