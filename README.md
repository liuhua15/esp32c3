#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdbool.h>
#include <unistd.h>
#include <time.h>
#include <sys/time.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/spi_master.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "nvs_flash.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "esp_sntp.h"
#include "protocol_examples_common.h"
#include "ssd1306.h"

// OLED配置
#define OLED_WIDTH 128
#define OLED_HEIGHT 64
#define OLED_MOSI 11
#define OLED_SCLK 12
#define OLED_DC 13
#define OLED_CS 14
#define OLED_RST 15
ssd1306_t display;

// 按键引脚
#define BTN_UP     2
#define BTN_DOWN   3
#define BTN_LEFT   4
#define BTN_RIGHT  5
#define BTN_A      6
#define BTN_B      7
#define BTN_SELECT 10
#define BTN_START  20

// 按键状态
bool btn_up, btn_down, btn_left, btn_right;
bool btn_a, btn_b, btn_select, btn_start;
bool last_up, last_down, last_left, last_right;
bool last_a, last_b, last_select, last_start;
// 系统状态
enum { 
    MENU_MAIN, MENU_CATEGORY, MENU_GAMES, GAME_PLAY, GAME_OVER,
    MENU_TOOLS, TOOL_COUNTDOWN_SET, TOOL_COUNTDOWN_RUN, TOOL_STOPWATCH,
    TOOL_CALENDAR, TOOL_TEMPERATURE, TOOL_ALARM_SET, TOOL_ALARM_RUN,
    MENU_SETTINGS, MENU_WIFI, MENU_TIME, MENU_ABOUT, MENU_HIGHSCORE
} state;

// 菜单位置
int menu_pos = 0;
int category = 0;
int game_idx = 0;
int tool_idx = 0;
int score = 0;
int highscore = 0;
int page = 0;

// 菜单文字
const char* main_menu[] = {"🎮 游戏", "🛠️ 工具", "⚙️ 设置", "📊 高分", "ℹ️ 关于"};
const char* categories[] = {"🔥 动作", "🧩 益智", "🏁 竞技", "🎲 经典"};
const char* tools_menu[] = {"⏱️ 倒计时", "⏲️ 秒表", "📅 日历", "🌡️ 温度", "⏰ 闹钟"};
const char* settings_menu[] = {"📶 WiFi", "🕒 时间", "💾 存档", "🔄 重置", "🔙 返回"};
// 28个游戏
const char* games[] = {
    // 动作游戏 (0-6)
    "🐍 贪吃蛇", "👾 太空侵略者", "🦖 恐龙跑酷", "🐦 Flappy Bird",
    "🧱 打砖块", "🏓 乒乓球", "🔫 枪战",
    
    // 益智游戏 (7-13)
    "🧩 俄罗斯方块", "🔢 2048", "⭕ 井字棋", "🧠 记忆挑战",
    "🔮 迷宫", "📦 推箱子", "🔢 数独",
    
    // 竞技游戏 (14-20)
    "🏎️ 赛车", "🏍️ 摩托GP", "🔄 漂移", "⛵ 快艇",
    "🚁 直升机", "✈️ 飞机", "🚀 太空赛车",
    
    // 经典游戏 (21-27)
    "🔨 打地鼠", "🎰 幸运转盘", "🎳 保龄球", "🏀 投篮",
    "⚽ 点球", "🎾 网球", "🥊 拳击"
};// WiFi配置（已修改）
const char* wifi_ssid = "CMCC-SgYd";
const char* wifi_pass = "73ttJd6r";

// 时间变量
int hour = 0, minute = 0, second = 0;
int year = 2024, month = 1, day = 1;
bool time_synced = false;
float temperature = 25.5;

// 工具变量
int countdown_value = 60, countdown_remain = 60;
bool countdown_run = false, countdown_pause = false;
unsigned long countdown_start = 0;

unsigned long stopwatch_val = 0;
bool stopwatch_run = false;
unsigned long stopwatch_start = 0;

int alarm_hour = 7, alarm_min = 0;
bool alarm_enable = false, alarm_trigger = false;

// 时间同步回调
void time_sync(struct timeval *tv) {
    time_synced = true;
    time_t now = time(NULL);
    struct tm *info = localtime(&now);
    hour = info->tm_hour;
    minute = info->tm_min;
    second = info->tm_sec;
    year = info->tm_year + 1900;
    month = info->tm_mon + 1;
    day = info->tm_mday;
}

// 初始化WiFi
void init_wifi() {
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    wifi_init_sta(wifi_ssid, wifi_pass);
}

// 初始化NTP
void init_ntp() {
    sntp_setoperatingmode(SNTP_OPMODE_POLL);
    sntp_setservername(0, "pool.ntp.org");
    sntp_set_time_sync_notification_cb(time_sync);
    sntp_init();
    for(int i=0; i<10 && !time_synced; i++) vTaskDelay(1000/portTICK_PERIOD_MS);
}
// 初始化OLED
void init_display() {
    spi_master_init(&display, OLED_MOSI, OLED_SCLK, OLED_DC, OLED_CS, OLED_RST);
    ssd1306_init(&display, OLED_WIDTH, OLED_HEIGHT);
    ssd1306_clear_screen(&display);
}

// 读按键
void read_buttons() {
    last_up = btn_up; last_down = btn_down;
    last_left = btn_left; last_right = btn_right;
    last_a = btn_a; last_b = btn_b;
    last_select = btn_select; last_start = btn_start;
    
    btn_up = !gpio_get_level(BTN_UP);
    btn_down = !gpio_get_level(BTN_DOWN);
    btn_left = !gpio_get_level(BTN_LEFT);
    btn_right = !gpio_get_level(BTN_RIGHT);
    btn_a = !gpio_get_level(BTN_A);
    btn_b = !gpio_get_level(BTN_B);
    btn_select = !gpio_get_level(BTN_SELECT);
    btn_start = !gpio_get_level(BTN_START);
}

bool pressed(bool cur, bool last) { return cur && !last; }
void show(const char* text, int x, int y) { ssd1306_display_text(&display, x, y, text, strlen(text), false); }
void clear() { ssd1306_clear_screen(&display); }
void rect(int x1, int y1, int x2, int y2, bool fill) { ssd1306_display_rectangle(&display, x1, y1, x2, y2, fill); }
void circle(int x, int y, int r, bool fill) { ssd1306_display_circle(&display, x, y, r, fill); }

// 显示头部
void header() {
    char buf[20];
    if(time_synced) sprintf(buf, "%02d:%02d:%02d", hour, minute, second);
    else sprintf(buf, "--:--:--");
    show(buf, 0, 0);
    sprintf(buf, "%.1fC", temperature);
    show(buf, 15, 0);
}
// 游戏数据
typedef struct {
    // 贪吃蛇
    int snake_x[100], snake_y[100], snake_len, food_x, food_y, dir;
    // 太空侵略者
    int invader_enemy[16], invader_x[16], invader_y[16], invader_bullet_x, invader_bullet_y, invader_gun;
    // 恐龙跑酷
    int dino_y, dino_vel, dino_obs_x;
    // Flappy Bird
    int flappy_y, flappy_vel, flappy_pipe_x, flappy_pipe_gap;
    // 打砖块
    int brick_ball_x, brick_ball_y, brick_ball_vx, brick_ball_vy, brick_paddle_x, brick_bricks[5][8];
    // 乒乓球
    int pong_ball_x, pong_ball_y, pong_ball_vx, pong_ball_vy, pong_paddle_y;
    // 枪战
    int gun_p1, gun_p2, gun_bullet;
    // 俄罗斯方块
    int tetris_grid[10][20], tetris_x, tetris_y, tetris_block;
    // 迷宫
    int maze_x, maze_y;
    // 推箱子
    int sokoban_px, sokoban_py, sokoban_box_x, sokoban_box_y;
    // 2048
    int grid2048[4][4];
    // 井字棋
    int tic_board[3][3], tic_turn, tic_cursor_x, tic_cursor_y;
    // 记忆
    int mem_seq[10], mem_step, mem_input;
    // 数独
    int sudoku_grid[9];
    // 赛车
    int race_car_x, race_road_y;
    // 直升机
    int heli_y;
    // 打地鼠
    int mole_x, mole_y;
    // 保龄球
    int bowl_power;
    // 投篮
    int basket_ball_y;
    // 点球
    int kick_ball_x;
    // 网球
    int tennis_ball_x, tennis_ball_y, tennis_vx, tennis_vy;
} game_data_t;

game_data_t g;// 初始化游戏
void init_game(int id) {
    memset(&g, 0, sizeof(g));
    score = 0;
    
    switch(id) {
        // 贪吃蛇
        case 0:
            g.snake_len = 3;
            g.snake_x[0] = 6; g.snake_y[0] = 4;
            g.snake_x[1] = 5; g.snake_y[1] = 4;
            g.snake_x[2] = 4; g.snake_y[2] = 4;
            g.dir = 1;
            g.food_x = 10; g.food_y = 4;
            break;
            
        // 太空侵略者
        case 1:
            for(int i=0; i<16; i++) {
                g.invader_enemy[i] = 1;
                g.invader_x[i] = 20 + (i%8)*10;
                g.invader_y[i] = 20 + (i/8)*8;
            }
            g.invader_gun = 60;
            g.invader_bullet_x = -1;
            break;
            
        // 恐龙跑酷
        case 2:
            g.dino_y = 40;
            g.dino_vel = 0;
            g.dino_obs_x = 128;
            break;
            
        // Flappy Bird
        case 3:
            g.flappy_y = 32;
            g.flappy_vel = 0;
            g.flappy_pipe_x = 128;
            g.flappy_pipe_gap = 30;
            break;
            
        // 打砖块
        case 4:
            g.brick_ball_x = 64; g.brick_ball_y = 40;
            g.brick_ball_vx = 2; g.brick_ball_vy = -2;
            g.brick_paddle_x = 54;
            for(int i=0;i<5;i++) for(int j=0;j<8;j++) g.brick_bricks[i][j]=1;
            break;
            
        // 乒乓球
        case 5:
            g.pong_ball_x = 60; g.pong_ball_y = 32;
            g.pong_ball_vx = 2; g.pong_ball_vy = 2;
            g.pong_paddle_y = 28;
            break;
            
        // 枪战
        case 6:
            g.gun_p1 = 20; g.gun_p2 = 100;
            g.gun_bullet = -1;
            break;
            
        // 俄罗斯方块
        case 7:
            memset(g.tetris_grid, 0, sizeof(g.tetris_grid));
            g.tetris_x = 4; g.tetris_y = 0;
            g.tetris_block = 1;
            break;
            
        // 2048
        case 8:
            memset(g.grid2048, 0, sizeof(g.grid2048));
            g.grid2048[0][0] = 2;
            g.grid2048[2][2] = 2;
            break;
            
        // 井字棋
        case 9:
            memset(g.tic_board, 0, sizeof(g.tic_board));
            g.tic_turn = 1;
            g.tic_cursor_x = 0; g.tic_cursor_y = 0;
            break;
            
        // 记忆挑战
        case 10:
            g.mem_step = 0;
            for(int i=0; i<5; i++) g.mem_seq[i] = rand()%4;
            break;
            
        // 迷宫
        case 11:
            g.maze_x = 10; g.maze_y = 10;
            break;
            
        // 推箱子
        case 12:
            g.sokoban_px = 2; g.sokoban_py = 2;
            g.sokoban_box_x = 4; g.sokoban_box_y = 4;
            break;
            
        // 数独
        case 13:
            for(int i=0;i<9;i++) g.sudoku_grid[i] = i+1;
            break;
            
        // 赛车
        case 14:
            g.race_car_x = 60;
            g.race_road_y = 0;
            break;
            
        // 摩托GP
        case 15:
            g.race_car_x = 60;
            g.race_road_y = 0;
            break;
            
        // 漂移
        case 16:
            g.race_car_x = 60;
            g.race_road_y = 0;
            break;
            
        // 快艇
        case 17:
            g.race_car_x = 60;
            g.race_road_y = 0;
            break;
            
        // 直升机
        case 18:
            g.heli_y = 32;
            break;
            
        // 飞机
        case 19:
            g.heli_y = 32;
            break;
            
        // 太空赛车
        case 20:
            g.race_car_x = 60;
            g.race_road_y = 0;
            break;
            
        // 打地鼠
        case 21:
            g.mole_x = 2; g.mole_y = 2;
            break;
            
        // 幸运转盘
        case 22:
            break;
            
        // 保龄球
        case 23:
            g.bowl_power = 0;
            break;
            
        // 投篮
        case 24:
            g.basket_ball_y = 50;
            break;
            
        // 点球
        case 25:
            g.kick_ball_x = 60;
            break;
            
        // 网球
        case 26:
            g.tennis_ball_x = 60; g.tennis_ball_y = 32;
            g.tennis_vx = 2; g.tennis_vy = 2;
            break;
            
        // 拳击
        case 27:
            break;
    }
}
// 更新游戏
void update_game(int id) {
    switch(id) {
        // 贪吃蛇
        case 0:
            if(pressed(btn_up, last_up) && g.dir != 2) g.dir = 0;
            if(pressed(btn_down, last_down) && g.dir != 0) g.dir = 2;
            if(pressed(btn_left, last_left) && g.dir != 1) g.dir = 3;
            if(pressed(btn_right, last_right) && g.dir != 3) g.dir = 1;
            
            int nx = g.snake_x[0], ny = g.snake_y[0];
            switch(g.dir) {
                case 0: ny--; break;
                case 1: nx++; break;
                case 2: ny++; break;
                case 3: nx--; break;
            }
            
            if(nx<0 || nx>=16 || ny<0 || ny>=8) { state = GAME_OVER; if(score>highscore) highscore=score; return; }
            for(int i=1; i<g.snake_len; i++) {
                if(g.snake_x[i]==nx && g.snake_y[i]==ny) { state = GAME_OVER; if(score>highscore) highscore=score; return; }
            }
            
            for(int i=g.snake_len-1; i>0; i--) {
                g.snake_x[i] = g.snake_x[i-1];
                g.snake_y[i] = g.snake_y[i-1];
            }
            g.snake_x[0] = nx; g.snake_y[0] = ny;
            
            if(g.snake_x[0]==g.food_x && g.snake_y[0]==g.food_y) {
                g.snake_len++; score += 10;
                g.food_x = rand()%16; g.food_y = rand()%8;
            }
            break;
            
        // 太空侵略者
        case 1:
            if(pressed(btn_left, last_left) && g.invader_gun>10) g.invader_gun -= 8;
            if(pressed(btn_right, last_right) && g.invader_gun<110) g.invader_gun += 8;
            if(pressed(btn_a, last_a) && g.invader_bullet_x==-1) {
                g.invader_bullet_x = g.invader_gun+4;
                g.invader_bullet_y = 52;
            }
            if(g.invader_bullet_y > 0) {
                g.invader_bullet_y -= 3;
                for(int i=0; i<16; i++) {
                    if(g.invader_enemy[i] && abs(g.invader_bullet_x-g.invader_x[i])<6 && abs(g.invader_bullet_y-g.invader_y[i])<6) {
                        g.invader_enemy[i] = 0;
                        g.invader_bullet_x = -1;
                        score += 10;
                    }
                }
                if(g.invader_bullet_y < 5) g.invader_bullet_x = -1;
            }
            int dead = 0;
            for(int i=0;i<16;i++) if(g.invader_enemy[i]==0) dead++;
            if(dead==16) { state = GAME_OVER; if(score>highscore) highscore=score; }
            break;
            
        // 恐龙跑酷
        case 2:
            if(pressed(btn_a, last_a) && g.dino_y==40) g.dino_vel = -8;
            g.dino_vel += 1;
            g.dino_y += g.dino_vel;
            if(g.dino_y > 40) g.dino_y = 40;
            g.dino_obs_x -= 5;
            if(g.dino_obs_x < 0) { g.dino_obs_x = 128; score += 10; }
            if(g.dino_obs_x<30 && g.dino_obs_x>20 && g.dino_y>30) { state = GAME_OVER; if(score>highscore) highscore=score; }
            break;
            
        // Flappy Bird
        case 3:
            if(pressed(btn_a, last_a)) g.flappy_vel = -4;
            g.flappy_vel += 2;
            g.flappy_y += g.flappy_vel;
            g.flappy_pipe_x -= 3;
            if(g.flappy_pipe_x < 0) {
                g.flappy_pipe_x = 128;
                g.flappy_pipe_gap = 20 + rand()%20;
                score += 10;
            }
            if(g.flappy_y<0 || g.flappy_y>60 || (g.flappy_pipe_x<50 && g.flappy_pipe_x>30 && (g.flappy_y<g.flappy_pipe_gap || g.flappy_y>g.flappy_pipe_gap+20))) {
                state = GAME_OVER; if(score>highscore) highscore=score;
            }
            break;
            
        // 打砖块
        case 4:
            if(pressed(btn_left, last_left)) g.brick_paddle_x -= 8;
            if(pressed(btn_right, last_right)) g.brick_paddle_x += 8;
            if(g.brick_paddle_x < 0) g.brick_paddle_x = 0;
            if(g.brick_paddle_x > 108) g.brick_paddle_x = 108;
            g.brick_ball_x += g.brick_ball_vx;
            g.brick_ball_y += g.brick_ball_vy;
            if(g.brick_ball_x<3 || g.brick_ball_x>124) g.brick_ball_vx = -g.brick_ball_vx;
            if(g.brick_ball_y<3) g.brick_ball_vy = -g.brick_ball_vy;
            if(g.brick_ball_y>53 && g.brick_ball_x>g.brick_paddle_x && g.brick_ball_x<g.brick_paddle_x+20) {
                g.brick_ball_vy = -g.brick_ball_vy;
            }
            if(g.brick_ball_y > 60) { state = GAME_OVER; if(score>highscore) highscore=score; }
            break;
    }
}// 乒乓球
        case 5:
            if(pressed(btn_up, last_up)) g.pong_paddle_y -= 5;
            if(pressed(btn_down, last_down)) g.pong_paddle_y += 5;
            if(g.pong_paddle_y < 0) g.pong_paddle_y = 0;
            if(g.pong_paddle_y > 56) g.pong_paddle_y = 56;
            g.pong_ball_x += g.pong_ball_vx;
            g.pong_ball_y += g.pong_ball_vy;
            if(g.pong_ball_y<=0 || g.pong_ball_y>=63) g.pong_ball_vy = -g.pong_ball_vy;
            if(g.pong_ball_x >= 120) {
                if(g.pong_ball_y>=g.pong_paddle_y && g.pong_ball_y<=g.pong_paddle_y+16) {
                    g.pong_ball_vx = -g.pong_ball_vx; score += 10;
                } else { g.pong_ball_x = 60; g.pong_ball_y = 32; score -= 5; }
            }
            if(g.pong_ball_x <= 0) { g.pong_ball_vx = -g.pong_ball_vx; score += 5; }
            break;
            
        // 枪战
        case 6:
            if(pressed(btn_a, last_a) && g.gun_bullet==-1) g.gun_bullet = g.gun_p1+5;
            if(g.gun_bullet > 0) g.gun_bullet += 5;
            if(g.gun_bullet > 120) g.gun_bullet = -1;
            if(g.gun_bullet > g.gun_p2-5 && g.gun_bullet < g.gun_p2+5) { state = GAME_OVER; if(score>highscore) highscore=score; }
            break;
    }
}

// 绘制游戏
void draw_game(int id) {
    clear();
    header();
    char buf[20];
    sprintf(buf, "得分:%d", score);
    show(buf, 0, 1);
    
    switch(id) {
        case 0: // 贪吃蛇
            for(int i=0; i<g.snake_len; i++) {
                rect(g.snake_x[i]*8, g.snake_y[i]*6+20, g.snake_x[i]*8+6, g.snake_y[i]*6+24, 1);
            }
            rect(g.food_x*8, g.food_y*6+20, g.food_x*8+6, g.food_y*6+24, 1);
            break;
            
        case 1: // 太空侵略者
            for(int i=0;i<16;i++) if(g.invader_enemy[i]) rect(g.invader_x[i], g.invader_y[i], g.invader_x[i]+6, g.invader_y[i]+4, 1);
            rect(g.invader_gun, 54, g.invader_gun+8, 58, 1);
            if(g.invader_bullet_x>0) rect(g.invader_bullet_x, g.invader_bullet_y, g.invader_bullet_x+2, g.invader_bullet_y+4, 1);
            break;
            
        case 2: // 恐龙
            rect(20, g.dino_y, 28, g.dino_y+12, 1);
            rect(g.dino_obs_x, 48, g.dino_obs_x+6, 56, 1);
            break;
            
        case 3: // Flappy
            circle(40, g.flappy_y, 3, 1);
            rect(g.flappy_pipe_x, 0, g.flappy_pipe_x+6, g.flappy_pipe_gap-10, 1);
            rect(g.flappy_pipe_x, g.flappy_pipe_gap+20, g.flappy_pipe_x+6, 64, 1);
            break;
            
        case 4: // 打砖块
            rect(g.brick_paddle_x, 56, g.brick_paddle_x+20, 60, 1);
            circle(g.brick_ball_x, g.brick_ball_y, 3, 1);
            for(int i=0;i<5;i++) for(int j=0;j<8;j++) if(g.brick_bricks[i][j]) rect(j*15+10, i*8+20, j*15+22, i*8+26, 1);
            break;
            
        case 5: // 乒乓球
            rect(120, g.pong_paddle_y, 128, g.pong_paddle_y+16, 1);
            circle(g.pong_ball_x, g.pong_ball_y, 3, 1);
            break;
            
        case 6: // 枪战
            rect(g.gun_p1, 30, g.gun_p1+5, 40, 1);
            rect(g.gun_p2, 30, g.gun_p2+5, 40, 1);
            if(g.gun_bullet>0) rect(g.gun_bullet, 33, g.gun_bullet+2, 37, 1);
            break;
            
        default:
            sprintf(buf, "玩: %s", games[id]);
            show(buf, 0, 3);
            break;
    }
    show("B:返回", 0, 7);
}// 显示主菜单
void show_main() {
    clear(); header();
    for(int i=0;i<5;i++) {
        char line[20];
        sprintf(line, "%s %s", menu_pos==i?">":" ", main_menu[i]);
        show(line, 0, 2+i);
    }
}

// 显示分类
void show_category() {
    clear(); header();
    show("选择分类", 0, 1);
    for(int i=0;i<4;i++) {
        char line[20];
        sprintf(line, "%s %s", category==i?">":" ", categories[i]);
        show(line, 0, 3+i);
    }
    show("A:进入 B:返回", 0, 7);
}

// 显示游戏列表
void show_games() {
    clear(); header();
    char line[20];
    sprintf(line, "%s", categories[category]);
    show(line, 0, 1);
    int start = category*7;
    for(int i=0;i<7 && start+i<28;i++) {
        sprintf(line, "%s %s", game_idx==i?">":" ", games[start+i]);
        show(line, 0, 3+i);
    }
    show("A:开始 B:返回", 0, 7);
}

// 显示工具
void show_tools() {
    clear(); header();
    show("工具", 0, 1);
    for(int i=0;i<5;i++) {
        char line[20];
        sprintf(line, "%s %s", tool_idx==i?">":" ", tools_menu[i]);
        show(line, 0, 3+i);
    }
    show("A:选择 B:返回", 0, 7);
}

// 倒计时
void show_countdown() {
    clear(); header();
    if(state == TOOL_COUNTDOWN_SET) {
        show("设置倒计时", 0, 2);
        char buf[20];
        sprintf(buf, "%d秒", countdown_value);
        show(buf, 40, 4);
        show("UP/DN:调 A:开始", 0, 7);
    } else {
        int m = countdown_remain/60;
        int s = countdown_remain%60;
        char buf[20];
        sprintf(buf, "%02d:%02d", m, s);
        show(buf, 40, 4);
        if(countdown_run) show("A:暂停 B:重置", 0, 7);
        else show("A:开始 B:重置", 0, 7);
    }
}

// 秒表
void show_stopwatch() {
    clear(); header();
    show("秒表", 0, 2);
    int total = stopwatch_val/1000;
    int m = total/60;
    int s = total%60;
    int ms = (stopwatch_val%1000)/10;
    char buf[20];
    sprintf(buf, "%02d:%02d.%02d", m, s, ms);
    show(buf, 30, 4);
    if(stopwatch_run) show("A:暂停 B:复位", 0, 7);
    else show("A:开始 B:复位", 0, 7);
}

// 日历
void show_calendar() {
    clear(); header();
    char date[30];
    sprintf(date, "%d年%d月%d日", year, month, day);
    show(date, 0, 3);
    const char* week[] = {"日","一","二","三","四","五","六"};
    time_t now = time(NULL);
    struct tm *info = localtime(&now);
    sprintf(date, "星期%s", week[info->tm_wday]);
    show(date, 0, 5);
    show("B:返回", 0, 7);
}

// 游戏结束
void show_gameover() {
    clear(); header();
    show("游戏结束", 30, 2);
    char buf[20];
    sprintf(buf, "得分:%d", score);
    show(buf, 30, 4);
    sprintf(buf, "最高:%d", highscore);
    show(buf, 30, 5);
    show("A:重来 B:菜单", 0, 7);
}
// 初始化硬件
void init_hardware() {
    nvs_flash_init();
    init_wifi();
    init_ntp();
    
    int pins[] = {BTN_UP, BTN_DOWN, BTN_LEFT, BTN_RIGHT, BTN_A, BTN_B, BTN_SELECT, BTN_START};
    for(int i=0;i<8;i++) {
        gpio_set_direction(pins[i], GPIO_MODE_INPUT);
        gpio_set_pull_mode(pins[i], GPIO_PULLUP_ONLY);
    }
    
    init_display();
    temperature += (rand()%20)/10.0;
}

// 主函数
void app_main() {
    init_hardware();
    state = MENU_MAIN;
    int cur_game = 0;
    
    while(1) {
        read_buttons();
        
        switch(state) {
            case MENU_MAIN:
                show_main();
                if(pressed(btn_up, last_up)) menu_pos = (menu_pos-1+5)%5;
                if(pressed(btn_down, last_down)) menu_pos = (menu_pos+1)%5;
                if(pressed(btn_a, last_a)) {
                    if(menu_pos==0) state = MENU_CATEGORY;
                    if(menu_pos==1) { state = MENU_TOOLS; tool_idx=0; }
                }
                break;
                
            case MENU_CATEGORY:
                show_category();
                if(pressed(btn_up, last_up)) category = (category-1+4)%4;
                if(pressed(btn_down, last_down)) category = (category+1)%4;
                if(pressed(btn_a, last_a)) { state = MENU_GAMES; game_idx=0; }
                if(pressed(btn_b, last_b)) state = MENU_MAIN;
                break;
                
            case MENU_GAMES:
                show_games();
                if(pressed(btn_up, last_up)) game_idx = (game_idx-1+7)%7;
                if(pressed(btn_down, last_down)) game_idx = (game_idx+1)%7;
                if(pressed(btn_a, last_a)) {
                    cur_game = category*7 + game_idx;
                    init_game(cur_game);
                    state = GAME_PLAY;
                }
                if(pressed(btn_b, last_b)) state = MENU_CATEGORY;
                break;
                
            case GAME_PLAY:
                update_game(cur_game);
                draw_game(cur_game);
                if(pressed(btn_b, last_b)) state = MENU_GAMES;
                break;
                
            case GAME_OVER:
                show_gameover();
                if(pressed(btn_a, last_a)) {
                    init_game(cur_game);
                    state = GAME_PLAY;
                }
                if(pressed(btn_b, last_b)) state = MENU_GAMES;
                break;
                
            case MENU_TOOLS:
                show_tools();
                if(pressed(btn_up, last_up)) tool_idx = (tool_idx-1+5)%5;
                if(pressed(btn_down, last_down)) tool_idx = (tool_idx+1)%5;
                if(pressed(btn_a, last_a)) {
                    if(tool_idx==0) state = TOOL_COUNTDOWN_SET;
                    if(tool_idx==1) { state = TOOL_STOPWATCH; stopwatch_val=0; stopwatch_run=0; }
                    if(tool_idx==2) state = TOOL_CALENDAR;
                }
                if(pressed(btn_b, last_b)) state = MENU_MAIN;
                break;
                
            case TOOL_COUNTDOWN_SET:
                show_countdown();
                if(pressed(btn_up, last_up)) countdown_value += 10;
                if(pressed(btn_down, last_down)) countdown_value -= 10;
                if(countdown_value<10) countdown_value=10;
                if(countdown_value>3600) countdown_value=3600;
                if(pressed(btn_a, last_a)) {
                    countdown_remain = countdown_value;
                    state = TOOL_COUNTDOWN_RUN;
                }
                break;
                
            case TOOL_COUNTDOWN_RUN:
                show_countdown();
                if(pressed(btn_a, last_a)) countdown_run = !countdown_run;
                if(pressed(btn_b, last_b)) {
                    countdown_remain = countdown_value;
                    countdown_run = false;
                }
                if(countdown_run && countdown_remain>0) {
                    static unsigned long last = 0;
                    if(millis()-last>=1000) {
                        countdown_remain--;
                        last = millis();
                    }
                }
                break;
                
            case TOOL_STOPWATCH:
                show_stopwatch();
                if(pressed(btn_a, last_a)) stopwatch_run = !stopwatch_run;
                if(pressed(btn_b, last_b)) { stopwatch_val=0; stopwatch_run=0; }
                if(pressed(btn_start, last_start)) state = MENU_TOOLS;
                if(stopwatch_run) stopwatch_val += 50;
                break;
                
            case TOOL_CALENDAR:
                show_calendar();
                if(pressed(btn_b, last_b)) state = MENU_TOOLS;
                break;
        }
        
        vTaskDelay(50/portTICK_PERIOD_MS);
    }
}
