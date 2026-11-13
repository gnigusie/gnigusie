name: Streak Stats
on:
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:

jobs:
  streak:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - name: Build streak card
        env:
          GH_TOKEN: ${{ secrets.METRICS_TOKEN }}
          GH_USER: gnigusie
        run: |
          python3 - <<'PY'
          import os, json, urllib.request
          from datetime import date, datetime, timedelta

          TOKEN = os.environ["GH_TOKEN"]
          USER = os.environ["GH_USER"]

          def gql(query, variables):
              req = urllib.request.Request(
                  "https://api.github.com/graphql",
                  data=json.dumps({"query": query, "variables": variables}).encode(),
                  headers={"Authorization": "bearer " + TOKEN,
                           "Content-Type": "application/json",
                           "User-Agent": "streak-card"},
              )
              with urllib.request.urlopen(req) as r:
                  data = json.load(r)
              if "errors" in data:
                  raise SystemExit(data["errors"])
              return data["data"]

          created = gql("query($u:String!){user(login:$u){createdAt}}", {"u": USER})["user"]["createdAt"]
          start_date = datetime.strptime(created[:10], "%Y-%m-%d").date()
          today = date.today()

          Q = """query($u:String!,$from:DateTime!,$to:DateTime!){
            user(login:$u){contributionsCollection(from:$from,to:$to){
              contributionCalendar{weeks{contributionDays{date contributionCount}}}}}}"""

          days = {}
          for y in range(start_date.year, today.year + 1):
              d = gql(Q, {"u": USER, "from": str(y) + "-01-01T00:00:00Z", "to": str(y) + "-12-31T23:59:59Z"})
              for w in d["user"]["contributionsCollection"]["contributionCalendar"]["weeks"]:
                  for c in w["contributionDays"]:
                      day = datetime.strptime(c["date"], "%Y-%m-%d").date()
                      if day.year == y:
                          days[day] = c["contributionCount"]

          all_days = sorted(k for k in days if k <= today)
          total = sum(days[k] for k in all_days)

          longest, run, run_start, l_start, l_end = 0, 0, None, None, None
          for k in all_days:
              if days[k] > 0:
                  if run == 0:
                      run_start = k
                  run += 1
                  if run > longest:
                      longest, l_start, l_end = run, run_start, k
              else:
                  run = 0

          d = today
          if days.get(d, 0) == 0:
              d -= timedelta(days=1)
          cur_end = d
          cur = 0
          while days.get(d, 0) > 0:
              cur += 1
              d -= timedelta(days=1)
          cur_start = d + timedelta(days=1)

          def fmt(x, year=True):
              return x.strftime("%b %-d, %Y" if year else "%b %-d")

          total_label = fmt(start_date) + " - Present"
          cur_label = (fmt(cur_start, False) + " - " + fmt(cur_end, False)) if cur > 0 else fmt(today, False)
          long_label = (fmt(l_start) + " - " + fmt(l_end)) if longest > 0 else "-"

          svg = f"""<svg xmlns="http://www.w3.org/2000/svg" width="495" height="195" viewBox="0 0 495 195" font-family="Segoe UI, Ubuntu, sans-serif" text-anchor="middle">
            <rect x="0.5" y="0.5" rx="4.5" width="494" height="194" fill="#141321" stroke="#e4e2e2" stroke-opacity="0.3"/>
            <line x1="165" y1="48" x2="165" y2="147" stroke="#e4e2e2" stroke-opacity="0.6"/>
            <line x1="330" y1="48" x2="330" y2="147" stroke="#e4e2e2" stroke-opacity="0.6"/>
            <text x="82" y="95" font-size="28" font-weight="700" fill="#7bf1a8">{total:,}</text>
            <text x="82" y="125" font-size="14" fill="#fe428e">Total Contributions</text>
            <text x="82" y="150" font-size="12" fill="#a9fef7">{total_label}</text>
            <circle cx="247" cy="82" r="40" fill="none" stroke="#7bf1a8" stroke-width="5"/>
            <text x="247" y="92" font-size="28" font-weight="700" fill="#f8d847">{cur}</text>
            <text x="247" y="150" font-size="14" font-weight="700" fill="#f8d847">Current Streak</text>
            <text x="247" y="170" font-size="12" fill="#a9fef7">{cur_label}</text>
            <text x="412" y="95" font-size="28" font-weight="700" fill="#7bf1a8">{longest}</text>
            <text x="412" y="125" font-size="14" fill="#fe428e">Longest Streak</text>
            <text x="412" y="150" font-size="12" fill="#a9fef7">{long_label}</text>
          </svg>"""

          with open("streak-stats.svg", "w", encoding="utf-8") as f:
              f.write(svg)
          PY
      - name: Commit card
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add streak-stats.svg
          git diff --staged --quiet || git commit -m "Update streak stats"
          git pull --rebase
          git push
