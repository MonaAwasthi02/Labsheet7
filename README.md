# Labsheet7 
1. Create usernames.txt with one GitHub username per line (or paste a single username).
2. Optionally set an environment variable GITHUB_TOKEN with a personal access token to increase rate limits.
3. Run: python github_scanner.py

import requests
import os
import time
from datetime import datetime

# CONFIG
INPUT_FILE = "usernames.txt"
OUTPUT_FILE = "github_report.txt"   # set to None to skip file output
GITHUB_API = "https://api.github.com"
# Optional: place your token in env var GITHUB_TOKEN or set here (not recommended to hardcode)
GITHUB_TOKEN = os.getenv("GITHUB_TOKEN", None)

HEADERS = {"Accept": "application/vnd.github+json"}
if GITHUB_TOKEN:
    HEADERS["Authorization"] = f"Bearer {GITHUB_TOKEN}"

# Helper: perform GET with basic rate-limit handling (simple)
def github_get(url, params=None):
    resp = requests.get(url, headers=HEADERS, params=params, timeout=15)
    if resp.status_code == 401:
        raise SystemExit("Unauthorized: check your GITHUB_TOKEN (if used).")
    if resp.status_code == 403 and "rate limit" in resp.text.lower():
        reset = resp.headers.get("X-RateLimit-Reset")
        if reset:
            wait = int(reset) - int(time.time())
            raise SystemExit(f"Rate limit exceeded. Retry after {wait} seconds.")
        else:
            raise SystemExit("Rate limit exceeded. Try using a token or wait.")
    resp.raise_for_status()
    return resp.json()

def scan_user(username):
    report = []
    report.append(f"--- Report for {username} ---")
    # 1) Basic user info
    try:
        user = github_get(f"{GITHUB_API}/users/{username}")
    except requests.HTTPError as e:
        report.append(f"Error fetching user {username}: {e}")
        return "\n".join(report)
    except SystemExit as e:
        return str(e)

    name = user.get("name") or "N/A"
    bio = user.get("bio") or "N/A"
    public_repos = user.get("public_repos", 0)
    followers = user.get("followers", 0)
    following = user.get("following", 0)
    created_at = user.get("created_at")
    created_at_fmt = created_at and datetime.fromisoformat(created_at.replace("Z", "+00:00")).strftime("%Y-%m-%d %H:%M:%S UTC")

    report.append(f"Name: {name}")
    report.append(f"Bio: {bio}")
    report.append(f"Public repos: {public_repos}")
    report.append(f"Followers: {followers} | Following: {following}")
    if created_at_fmt:
        report.append(f"Account created: {created_at_fmt}")

    # 2) List repositories (first page; you can extend pagination if needed)
    repos = []
    page = 1
    per_page = 100
    collected = 0
    # we'll paginate through until we have all repos (safe for public scans)
    while True:
        params = {"per_page": per_page, "page": page, "sort": "updated"}
        repo_page = github_get(f"{GITHUB_API}/users/{username}/repos", params=params)
        if not isinstance(repo_page, list):
            break
        repos.extend(repo_page)
        collected += len(repo_page)
        if len(repo_page) < per_page:
            break
        page += 1

    if not repos:
        report.append("No public repositories found.")
        return "\n".join(report)

    report.append(f"Found {len(repos)} public repositories (showing summary):")

    total_stars = 0
    total_forks = 0
    languages_seen = {}

    # Summarize each repo
    for r in repos:
        name_r = r.get("name")
        stars = r.get("stargazers_count", 0)
        forks = r.get("forks_count", 0)
        language = r.get("language") or "None"
        updated = r.get("updated_at")
        updated_fmt = updated and datetime.fromisoformat(updated.replace("Z", "+00:00")).strftime("%Y-%m-%d")
        repo_line = f"  - {name_r}: ⭐{stars} | 🍴{forks} | language={language} | updated={updated_fmt}"
        report.append(repo_line)
        total_stars += stars
        total_forks += forks
        languages_seen[language] = languages_seen.get(language, 0) + 1

    report.append(f"Total stars: {total_stars} | Total forks: {total_forks}")
    # Top languages summary
    lang_summary = ", ".join(f"{lang}:{count}" for lang, count in sorted(languages_seen.items(), key=lambda x: -x[1]))
    report.append(f"Languages used (count of repos): {lang_summary}")

    return "\n".join(report)


def main():
    if not os.path.exists(INPUT_FILE):
        print(f"Input file '{INPUT_FILE}' not found. Create it and add GitHub usernames (one per line).")
        return

    with open(INPUT_FILE, "r", encoding="utf-8") as f:
        lines = [line.strip() for line in f if line.strip() and not line.strip().startswith("#")]

    if not lines:
        print(f"No usernames found in {INPUT_FILE}. Put one username per line.")
        return

    all_reports = []
    for username in lines:
        print(f"Scanning {username} ...")
        rep = scan_user(username)
        print(rep)
        all_reports.append(rep)
        # small delay to be polite and reduce risk of rate limit
        time.sleep(1)

    if OUTPUT_FILE:
        with open(OUTPUT_FILE, "w", encoding="utf-8") as out:
            out.write("\n\n".join(all_reports))
        print(f"\nSaved full report to {OUTPUT_FILE}")


if __name__ == "__main__":
    main()
