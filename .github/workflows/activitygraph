"""Draws a 'last 31 days' contribution graph as an SVG, using GitHub's own data.

Runs inside the profile workflow, so it never depends on a public card service.
Usage: python activity_graph.py <github-login> <output.svg>
Needs the GITHUB_TOKEN environment variable (the workflow's default token is enough).
Set ACTIVITY_SAMPLE=1 to draw made-up data for a local preview.
"""
import datetime as dt
import json
import os
import sys
import urllib.request

DAYS = 31
W, H = 900, 300
PAD_L, PAD_R, PAD_T, PAD_B = 56, 28, 64, 44

BG, GRID, TEXT, MUTED = "#0d1117", "#21262d", "#e6edf3", "#8b949e"
ACCENT = "#e5484d"

QUERY = """
query($login: String!, $from: DateTime!, $to: DateTime!) {
  user(login: $login) {
    contributionsCollection(from: $from, to: $to) {
      contributionCalendar { weeks { contributionDays { date contributionCount } } }
    }
  }
}
"""


def fetch(login):
    today = dt.datetime.now(dt.timezone.utc).date()
    start = today - dt.timedelta(days=DAYS - 1)
    if os.environ.get("ACTIVITY_SAMPLE"):
        import random
        random.seed(3)
        return [(start + dt.timedelta(days=i), random.choice([0, 0, 0, 1, 2, 4, 7, 12]))
                for i in range(DAYS)]
    body = json.dumps({"query": QUERY, "variables": {
        "login": login,
        "from": f"{start.isoformat()}T00:00:00Z",
        "to": f"{today.isoformat()}T23:59:59Z",
    }}).encode()
    req = urllib.request.Request("https://api.github.com/graphql", data=body, headers={
        "Authorization": f"bearer {os.environ['GITHUB_TOKEN']}",
        "Content-Type": "application/json",
        "User-Agent": "profile-activity-graph",
    })
    with urllib.request.urlopen(req, timeout=30) as r:
        data = json.load(r)
    if "errors" in data:
        raise SystemExit(f"GitHub API error: {data['errors']}")
    counts = {}
    for week in data["data"]["user"]["contributionsCollection"]["contributionCalendar"]["weeks"]:
        for d in week["contributionDays"]:
            counts[d["date"]] = d["contributionCount"]
    return [(start + dt.timedelta(days=i), counts.get((start + dt.timedelta(days=i)).isoformat(), 0))
            for i in range(DAYS)]


def nice_max(v):
    for step in (1, 2, 5, 10, 20, 25, 50, 100, 200, 500):
        if v <= step * 4:
            return step * 4, step
    return v, max(1, v // 4)


def smooth_path(pts):
    """Monotone-ish smooth curve through the points (Catmull-Rom to Bezier, clamped)."""
    d = f"M{pts[0][0]:.1f},{pts[0][1]:.1f}"
    for i in range(len(pts) - 1):
        p0 = pts[i - 1] if i else pts[i]
        p1, p2 = pts[i], pts[i + 1]
        p3 = pts[i + 2] if i + 2 < len(pts) else p2
        c1 = (p1[0] + (p2[0] - p0[0]) / 6, p1[1] + (p2[1] - p0[1]) / 6)
        c2 = (p2[0] - (p3[0] - p1[0]) / 6, p2[1] - (p3[1] - p1[1]) / 6)
        lo, hi = min(p1[1], p2[1]), max(p1[1], p2[1])
        c1 = (c1[0], min(max(c1[1], lo), hi))
        c2 = (c2[0], min(max(c2[1], lo), hi))
        d += f" C{c1[0]:.1f},{c1[1]:.1f} {c2[0]:.1f},{c2[1]:.1f} {p2[0]:.1f},{p2[1]:.1f}"
    return d


def render(days):
    total = sum(c for _, c in days)
    active = sum(1 for _, c in days if c)
    best_day, best = max(days, key=lambda x: x[1])
    ymax, step = nice_max(max(best, 1))
    pw, ph = W - PAD_L - PAD_R, H - PAD_T - PAD_B
    x = lambda i: PAD_L + pw * i / (len(days) - 1)
    y = lambda v: PAD_T + ph * (1 - v / ymax)
    pts = [(x(i), y(c)) for i, (_, c) in enumerate(days)]
    line = smooth_path(pts)
    area = f"{line} L{pts[-1][0]:.1f},{PAD_T + ph} L{pts[0][0]:.1f},{PAD_T + ph} Z"

    font = "font-family=\"-apple-system,BlinkMacSystemFont,'Segoe UI',Helvetica,Arial,sans-serif\""
    out = [f'<svg xmlns="http://www.w3.org/2000/svg" width="{W}" height="{H}" viewBox="0 0 {W} {H}" {font}>',
           '<defs><linearGradient id="fill" x1="0" y1="0" x2="0" y2="1">'
           f'<stop offset="0" stop-color="{ACCENT}" stop-opacity="0.35"/>'
           f'<stop offset="1" stop-color="{ACCENT}" stop-opacity="0"/></linearGradient></defs>',
           f'<rect width="{W}" height="{H}" rx="10" fill="{BG}"/>',
           f'<text x="{PAD_L}" y="32" fill="{TEXT}" font-size="17" font-weight="600">Contribution activity</text>',
           f'<text x="{PAD_L}" y="50" fill="{MUTED}" font-size="12">Last {DAYS} days · updated {dt.date.today():%d %b %Y}</text>',
           f'<text x="{W - PAD_R}" y="32" fill="{ACCENT}" font-size="17" font-weight="600" text-anchor="end">{total} contributions</text>',
           f'<text x="{W - PAD_R}" y="50" fill="{MUTED}" font-size="12" text-anchor="end">{active} active days · best {best} on {best_day:%d %b}</text>']
    v = 0
    while v <= ymax:
        yy = y(v)
        out.append(f'<line x1="{PAD_L}" x2="{W - PAD_R}" y1="{yy:.1f}" y2="{yy:.1f}" stroke="{GRID}"/>')
        out.append(f'<text x="{PAD_L - 10}" y="{yy + 4:.1f}" fill="{MUTED}" font-size="11" text-anchor="end">{v}</text>')
        v += step
    for i, (d, _) in enumerate(days):
        if i % 5 == 0 or i == len(days) - 1:
            out.append(f'<text x="{x(i):.1f}" y="{H - 18}" fill="{MUTED}" font-size="11" text-anchor="middle">{d:%d %b}</text>')
    out.append(f'<path d="{area}" fill="url(#fill)"/>')
    out.append(f'<path d="{line}" fill="none" stroke="{ACCENT}" stroke-width="2.5" stroke-linejoin="round"/>')
    for (px, py), (_, c) in zip(pts, days):
        if c:
            out.append(f'<circle cx="{px:.1f}" cy="{py:.1f}" r="3.5" fill="{BG}" stroke="{ACCENT}" stroke-width="2"/>')
    out.append("</svg>")
    return "\n".join(out)


if __name__ == "__main__":
    login, path = sys.argv[1], sys.argv[2]
    os.makedirs(os.path.dirname(path) or ".", exist_ok=True)
    with open(path, "w") as f:
        f.write(render(fetch(login)))
    print(f"wrote {path}")
