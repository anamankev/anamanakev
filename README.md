#!/usr/bin/env python3
"""
Generates a neofetch-style GitHub profile card: personal ASCII portrait on the
left, live account stats + personal fields on the right.

Usage:
    python3 generate_card.py            # DEMO_MODE if no GH_TOKEN set
    GH_TOKEN=ghp_xxx python3 generate_card.py   # pulls real live stats

Requires: pillow, numpy, PyGithub, requests   (see requirements.txt)
"""

import os
import sys
import datetime
import numpy as np
from PIL import Image, ImageOps, ImageFilter, ImageDraw, ImageFont

# ---------------------------------------------------------------------------
# EDIT THIS BLOCK: your own static fields. Nothing here is pulled from an
# API, it's just you. Change any of it freely.
# ---------------------------------------------------------------------------
PROFILE = {
    "user_at_host": "kelvin@pulse",
    "os": "macOS, Linux",
    "host": "Pulse Organization",
    "kernel": "Founder & CEO",
    "ide": "VS Code",
    "languages_programming": "Python, C++, TypeScript, C",
    "languages_computer": "HTML, CSS, LaTeX, YAML",
    "languages_real": "English",
    "hobbies_software": "Competitive Math, Research",
    "hobbies_hardware": "Grid Systems, Embedded Firmware",
    "email": "YOUR_EMAIL",
    "linkedin": "YOUR_LINKEDIN",
}

GITHUB_USERNAME = os.getenv("GH_USERNAME", "YOUR_GITHUB_USERNAME")

# ---------------------------------------------------------------------------
# Visual theme
# ---------------------------------------------------------------------------
BG = (11, 12, 20)
ART_COLOR = (154, 194, 255)
HEADER_COLOR = (253, 184, 19)     # amber accent, matches the README badges
LABEL_COLOR = (122, 162, 247)     # tokyonight blue
VALUE_COLOR = (192, 202, 245)     # soft off-white
DIM_COLOR = (90, 96, 120)
FONT_PATH = "/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf"
FONT_BOLD_PATH = "/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf"


# ---------------------------------------------------------------------------
# 1. ASCII portrait from assets/avatar.jpg
# ---------------------------------------------------------------------------
def ascii_portrait(image_path, out_width=32):
    ramp = " .:-=+*#%@"
    full = Image.open(image_path).convert("L")
    w, h = full.size
    left, right = int(w * 0.20), int(w * 0.84)
    top, bottom = 0, int(h * 0.86)
    cropped = full.crop((left, top, right, bottom))

    cw, ch = cropped.size
    out_height = max(1, int(out_width * (ch / cw) * 0.48))

    base = ImageOps.autocontrast(cropped, cutoff=2).resize((out_width, out_height), Image.LANCZOS)
    base_arr = np.asarray(base).astype(np.float32)

    edge = cropped.filter(ImageFilter.GaussianBlur(1.2)).filter(ImageFilter.FIND_EDGES)
    edge = ImageOps.autocontrast(edge, cutoff=1).resize((out_width, out_height), Image.LANCZOS)
    edge_arr = np.asarray(edge).astype(np.float32)

    combo = np.clip(base_arr * 0.72 + edge_arr * 0.55, 0, 255)
    combo = (combo - combo.min()) / (combo.max() - combo.min() + 1e-6) * 255
    idx = (combo / 255 * (len(ramp) - 1)).astype(int)
    return ["".join(ramp[i] for i in row) for row in idx]


# ---------------------------------------------------------------------------
# 2. Live GitHub stats (falls back to demo placeholders with no token)
# ---------------------------------------------------------------------------
def fetch_stats(username):
    token = os.getenv("GH_TOKEN")
    if not token:
        print("no GH_TOKEN set, using placeholder demo stats", file=sys.stderr)
        return {
            "uptime": "-- years, -- days",
            "repos": "--",
            "commits": "--",
            "stars": "--",
            "followers": "--",
            "loc": "-- (++--, --)",
        }

    import requests
    from github import Github

    g = Github(token)
    u = g.get_user(username)

    created = u.created_at.replace(tzinfo=datetime.timezone.utc)
    now = datetime.datetime.now(datetime.timezone.utc)
    delta = now - created
    years, days = divmod(delta.days, 365)
    uptime = f"{years} years, {days} days"

    owned_repos = [r for r in u.get_repos(type="owner") if not r.fork]
    repos = len(owned_repos)
    stars = sum(r.stargazers_count for r in owned_repos)
    followers = u.followers

    # lifetime commit contributions via GraphQL, summed year by year
    headers = {"Authorization": f"bearer {token}"}
    total_commits = 0
    year_start = created
    while year_start < now:
        year_end = min(year_start + datetime.timedelta(days=365), now)
        query = """
        query($login: String!, $from: DateTime!, $to: DateTime!) {
          user(login: $login) {
            contributionsCollection(from: $from, to: $to) {
              totalCommitContributions
              restrictedContributionsCount
            }
          }
        }"""
        variables = {
            "login": username,
            "from": year_start.isoformat(),
            "to": year_end.isoformat(),
        }
        try:
            r = requests.post(
                "https://api.github.com/graphql",
                json={"query": query, "variables": variables},
                headers=headers,
                timeout=15,
            )
            data = r.json()["data"]["user"]["contributionsCollection"]
            total_commits += data["totalCommitContributions"] + data["restrictedContributionsCount"]
        except Exception as e:
            print(f"contributionsCollection error: {e}", file=sys.stderr)
        year_start = year_end

    # lines of code across owned repos via contributor stats (best-effort,
    # GitHub caches this and can return 202 while it computes)
    import time
    additions = deletions = 0
    for r in owned_repos[:60]:  # safety cap
        try:
            url = f"https://api.github.com/repos/{r.full_name}/stats/contributors"
            for attempt in range(3):
                resp = requests.get(url, headers=headers, timeout=15)
                if resp.status_code == 202:
                    time.sleep(2)
                    continue
                break
            if resp.status_code != 200:
                continue
            for contributor in resp.json():
                if contributor.get("author") and contributor["author"].get("login") == username:
                    for week in contributor.get("weeks", []):
                        additions += week.get("a", 0)
                        deletions += week.get("d", 0)
        except Exception as e:
            print(f"stats/contributors error on {r.full_name}: {e}", file=sys.stderr)

    loc = f"{additions - deletions} (++{additions}, --{deletions})"

    return {
        "uptime": uptime,
        "repos": str(repos),
        "commits": f"{total_commits:,}",
        "stars": str(stars),
        "followers": str(followers),
        "loc": loc,
    }


# ---------------------------------------------------------------------------
# 3. Compose the card
# ---------------------------------------------------------------------------
def build_lines(stats):
    L = []
    L.append(("header", PROFILE["user_at_host"]))
    L.append(("rule", "-" * len(PROFILE["user_at_host"])))
    L.append(("kv", "OS", PROFILE["os"]))
    L.append(("kv", "Uptime", stats["uptime"]))
    L.append(("kv", "Host", PROFILE["host"]))
    L.append(("kv", "Kernel", PROFILE["kernel"]))
    L.append(("kv", "IDE", PROFILE["ide"]))
    L.append(("kv", "Languages.Programming", PROFILE["languages_programming"]))
    L.append(("kv", "Languages.Computer", PROFILE["languages_computer"]))
    L.append(("kv", "Languages.Real", PROFILE["languages_real"]))
    L.append(("kv", "Hobbies.Software", PROFILE["hobbies_software"]))
    L.append(("kv", "Hobbies.Hardware", PROFILE["hobbies_hardware"]))
    L.append(("blank", ""))
    L.append(("section", "Contact"))
    L.append(("kv", "Email", PROFILE["email"]))
    L.append(("kv", "LinkedIn", PROFILE["linkedin"]))
    L.append(("blank", ""))
    L.append(("section", "GitHub Stats"))
    L.append(("kv", "Repos", stats["repos"]))
    L.append(("kv", "Commits", stats["commits"]))
    L.append(("kv", "Stars", stats["stars"]))
    L.append(("kv", "Followers", stats["followers"]))
    L.append(("kv", "Lines of Code", stats["loc"]))
    return L


def render(ascii_lines, info_lines, out_path, font_size=14):
    font = ImageFont.truetype(FONT_PATH, font_size)
    font_bold = ImageFont.truetype(FONT_BOLD_PATH, font_size)
    cw = font.getbbox("M")[2] + 1
    ch = font_size + 6
    pad = 28
    gap = 40

    art_w = cw * max(len(l) for l in ascii_lines)
    art_h = ch * len(ascii_lines)
    info_w = 560
    info_h = ch * len(info_lines)

    W = pad * 2 + art_w + gap + info_w
    H = pad * 2 + max(art_h, info_h)

    img = Image.new("RGB", (W, H), BG)
    d = ImageDraw.Draw(img)

    for i, line in enumerate(ascii_lines):
        d.text((pad, pad + i * ch), line, font=font, fill=ART_COLOR)

    x0 = pad + art_w + gap
    y = pad
    for entry in info_lines:
        kind = entry[0]
        if kind == "header":
            d.text((x0, y), entry[1], font=font_bold, fill=HEADER_COLOR)
        elif kind == "rule":
            d.text((x0, y), entry[1], font=font, fill=DIM_COLOR)
        elif kind == "section":
            d.text((x0, y), entry[1], font=font_bold, fill=HEADER_COLOR)
            d.line((x0, y + ch - 4, x0 + font_bold.getbbox(entry[1])[2], y + ch - 4), fill=HEADER_COLOR)
        elif kind == "kv":
            label, value = entry[1], entry[2]
            d.text((x0, y), f"{label}:", font=font, fill=LABEL_COLOR)
            d.text((x0 + cw * (len(label) + 2), y), value, font=font, fill=VALUE_COLOR)
        y += ch

    img.save(out_path)
    return out_path


if __name__ == "__main__":
    art = ascii_portrait("assets/avatar.jpg" if os.path.exists("assets/avatar.jpg") else "/mnt/user-data/uploads/IMG_5121.jpeg")
    stats = fetch_stats(GITHUB_USERNAME)
    lines = build_lines(stats)
    out = render(art, lines, "card.png")
    print(f"wrote {out}")
