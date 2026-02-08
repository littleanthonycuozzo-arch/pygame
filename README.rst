import os, sys, random, time
"""
X-TRACKER: RIFT - ELITE EDITION
A tactical terminal-based roguelike with combo systems and rift mechanics.
Author: [Your Name/GitHub Handle]
Version: 4.5
"""

import os
import sys
import random
import time

# --- CROSS-PLATFORM INPUT HANDLING ---
try:
    import msvcrt
    def get_key():
        k = msvcrt.getch().lower()
        return k.decode() if isinstance(k, bytes) else k
except ImportError:
    import tty, termios
    def get_key():
        fd = sys.stdin.fileno()
        old_settings = termios.tcgetattr(fd)
        try:
            tty.setraw(sys.stdin.fileno())
            ch = sys.stdin.read(1).lower()
        finally:
            termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)
        return ch

# --- COLORS & UI CONSTANTS ---
K = {
    "B": "\033[94m", "C": "\033[96m", "G": "\033[92m", "Y": "\033[93m", 
    "R": "\033[91m", "BG": "\033[44m", "W": "\033[97m", "E": "\033[0m", 
    "F": "\033[1m", "P": "\033[95m"
}

class Game:
    def __init__(self, mode):
        self.m, self.p, self.s, self.a, self.t, self.dead_h = mode, [3, 10], 0, True, [], []
        self.turn = 0
        self.kill_log = []
        self.combo = False
        self.bullet_path = []
        
        h_nums = {"EASY": (2, 5), "MEDIUM": (12, 18), "HARD": (30, 45)}
        count = random.randint(*h_nums[mode])
        self.h = [[pos, random.random() < 0.5] for pos in self.gen(count, True)]
        self.c = self.gen(5, False)

    def gen(self, n, is_h):
        items, buf = [], (1 if self.m == "HARD" else 2)
        while len(items) < n:
            pos = [random.randint(0, 6), random.randint(0, 10 if is_h else 9)]
            if pos != [3, 0] and (not is_h or (abs(pos[0]-3) > buf or abs(pos[1]-10) > buf)):
                if pos not in items: items.append(pos)
        return items

    def shoot(self, direction_input):
        if not self.h: return f"{K['C']}NO TARGETS LEFT!{K['E']}"
        
        dx = (1 if 'd' in direction_input else -1 if 'a' in direction_input else 0)
        dy = (1 if 'w' in direction_input else -1 if 's' in direction_input else 0)
        
        self.bullet_path = []
        for i in range(1, 12):
            bx, by = self.p[0] + (dx * i), self.p[1] + (dy * i)
            if 0 <= bx <= 6 and 0 <= by <= 10: self.bullet_path.append([bx, by])
            else: break
        
        self.render_frame(f"{K['W']}>>> FIRING!{K['E']}")
        time.sleep(0.12)
        self.bullet_path = [] 

        targets = [hd for hd in self.h if (1 if hd[0][0]>self.p[0] else -1 if hd[0][0]<self.p[0] else 0) == dx and (1 if hd[0][1]>self.p[1] else -1 if hd[0][1]<self.p[1] else 0) == dy]
        
        if not targets: return f"{K['R']}MISS - NO TARGET{K['E']}"
        target_data = min(targets, key=lambda d: (abs(d[0][0]-self.p[0]) + abs(d[0][1]-self.p[1])))
        
        if random.random() < 0.5:
            self.dead_h.append(list(target_data[0]))
            self.h.remove(target_data)
            self.kill_log.append(self.turn)
            if len(self.kill_log) >= 3 and self.kill_log[-1] - self.kill_log[-3] <= 5:
                self.combo = True
            return f"{K['G']}HIT! NEUTRALIZED{K['E']}"
        return f"{K['R']}MISS!{K['E']}"

    def get_rank(self):
        if self.s >= 200: return f"{K['P']}RIFT LEGEND{K['E']}"
        if self.s >= 100: return f"{K['C']}ELITE TRACKER{K['E']}"
        if self.s >= 40: return f"{K['G']}STRIKER{K['E']}"
        return f"{K['R']}FLEEING COWARD{K['E']}"

    def render_frame(self, msg=""):
        os.system('cls' if os.name == 'nt' else 'clear')
        combo_txt = f" {K['P']}{K['F']}COMBO ACTIVE (3x Coins)!{K['E']}" if self.combo else ""
        head = (f"{K['B']}┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓\n"
                f"┃{K['E']} {K['F']}MODE:{K['E']} {self.m:<6} {K['B']}|{K['E']} {K['F']}SCORE:{K['E']} {K['Y']}{self.s:03}{K['E']} {K['B']}┃\n"
                f"┃{K['E']} {K['F']}HUNTERS:{K['E']} {K['R']}{len(self.h):02}{K['E']} {combo_txt:<20} {K['B']}┃\n"
                f"┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛{K['E']}")
        
        rows = []
        for z in range(10, -1, -1):
            r = ["·"] * 7
            for i in self.c: (z==i[1] and r.__setitem__(i[0], f"{K['Y']}${K['E']}"))
            for i in self.t: (z==i[1] and r.__setitem__(i[0], f"{K['R']}_{K['E']}"))
            for i in self.h: (z==i[0][1] and r.__setitem__(i[0][0], f"{K['R']}!{K['E']}"))
            for i in self.dead_h: (z==i[1] and r.__setitem__(i[0], f"{K['P']}#{K['E']}"))
            for b in self.bullet_path: (z==b[1] and r.__setitem__(b[0], f"{K['W']}*{K['E']}"))
            if z == 0: r[3] = f"{K['G']}@{K['E']}"
            if z == self.p[1]: r[self.p[0]] = f"{K['C']}{K['F']}X{K['E']}"
            rows.append(f"          {K['B']}|{K['E']} {' '.join(r)} {K['B']}|{K['E']}")
        print(head + "\n" + "\n".join(rows) + f"\n{msg}")

    def play(self):
        msg = ""
        while self.a:
            self.render_frame(msg + f"\n{K['C']}ACTION >> {K['E']}")
            inp = input().lower()
            self.turn += 1
            msg = ""
            if 'f' in inp: msg = self.shoot(inp)
            else:
                new_p = list(self.p)
                if 'w' in inp: new_p[1] = min(10, new_p[1]+1)
                if 's' in inp: new_p[1] = max(0, new_p[1]-1)
                if 'a' in inp: new_p[0] = max(0, new_p[0]-1)
                if 'd' in inp: new_p[0] = min(6, new_p[0]+1)
                
                dist = abs(new_p[0]-3) + abs(new_p[1]-0)
                if new_p == [3, 0]:
                    print(f"\n{K['G']}ESCAPED! SCORE: {self.s}{K['E']}")
                    print(f"RANK: {self.get_rank()}")
                    self.a = False; break
                elif dist == 1:
                    self.p = [new_p[0], new_p[1] + 1]
                    for pos in self.gen(len(self.h), True): self.h.append([pos, False])
                    for hd in self.h: hd[1] = False
                    msg = f"{K['R']}RIFT TRIGGERED!{K['E']}"
                elif new_p in self.dead_h: msg = f"{K['Y']}BLOCKED!{K['E']}"
                else: self.p = new_p

            # Hunter Logic
            for hd in self.h:
                h, scav = hd
                t = self.p if (not scav or not self.c) else min(self.c, key=lambda c: abs(c[0]-h[0])+abs(c[1]-h[1]))
                for i in (0,1):
                    if h[i] != t[i]: h[i] += 1 if t[i] > h[i] else -1
                if scav and h in self.c: self.c.remove(h)

            # Hazard Conversion
            locs = {}
            for hd in self.h: locs[tuple(hd[0])] = locs.get(tuple(hd[0]), 0) + 1
            self.h = [hd for hd in self.h if locs[tuple(hd[0])] == 1 or [self.t.append(list(hd[0])) and False]]
            
            if self.p in [h[0] for h in self.h] or self.p in self.t:
                print(f"\n{K['R']}WASTED! FINAL SCORE: {self.s}{K['E']}\nRANK: {K['R']}DEAD MEAT{K['E']}")
                self.a = False
            
            for coin in list(self.c):
                if self.p == coin:
                    self.s += 30 if self.combo else 10
                    self.c.remove(coin)
                    self.combo = False 
        input("\n[PRESS ENTER TO EXIT]")

def main_menu():
    m, s, opts = "EASY", 0, ["START", "MODE", "HOW TO", "EXIT"]
    while True:
        os.system('cls' if os.name == 'nt' else 'clear')
        print(f"{K['B']}┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓\n┃{K['E']}        {K['C']}{K['F']}X-TRACKER: ELITE{K['E']}           {K['B']}┃\n┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫{K['E']}")
        for i, o in enumerate(opts):
            lbl = o if o != 'MODE' else f"MODE: {m}"
            txt = f" > {lbl}"; pad = " " * (34 - len(txt))
            clr = K['BG']+K['W'] if i == s else ""
            print(f"{K['B']}┃{K['E']} {clr}{txt}{pad}{K['E']} {K['B']}┃{K['E']}")
        print(f"{K['B']}┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛\n {K['C']}[A/D] Move [S] Select{K['E']}")
        
        k = get_key()
        if k == 'a': s = (s - 1) % len(opts)
        elif k == 'd': s = (s + 1) % len(opts)
        elif k == 's':
            if s == 0: Game(m).play()
            elif s == 1: m = ["EASY", "MEDIUM", "HARD"][(["EASY", "MEDIUM", "HARD"].index(m)+1)%3]
            elif s == 2:
                print(f"\n{K['P']}COMBO:{K['E']} 3 kills in 5 turns = 3x Coins.\n{K['W']}TRACE:{K['E']} Shots show path.\n{K['G']}RANK:{K['E']} Get out alive to earn status.")
                input("\n[ENTER]")
            elif s == 3: break

if __name__ == "__main__":
    main_menu()