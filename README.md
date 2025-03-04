# standalone-Application-for-API-Communication-between-Jira-Bot-and-Slack-Bot
import os
import time
import requests
import json
from dotenv import load_dotenv
from requests.auth import HTTPBasicAuth
from datetime import datetime

# Load environment variables
load_dotenv()

JIRA_URL = "https://vineetinamati.atlassian.net"
JIRA_EMAIL = os.getenv("JIRA_EMAIL")
JIRA_API_TOKEN = os.getenv("JIRA_API_TOKEN")
JIRA_PROJECT = os.getenv("JIRA_PROJECT")

SLACK_TOKEN = os.getenv("SLACK_TOKEN")
SLACK_USER_ID = os.getenv("SLACK_USER_ID")

JIRA_API_ENDPOINT = f"{JIRA_URL}/rest/api/3/search?jql=project={JIRA_PROJECT}&fields=key,summary,status,updated"
HEADERS = {
    "Accept": "application/json",
    "Content-Type": "application/json"
}

AUTH = HTTPBasicAuth("vineetinamati@gmail.com", "ATATT3xFfGF024vqVwtWImIRI2SPIJy6J5MAGMwG7bP52CYIFeRIdNLasof7ttiT9t0ZYaIxl05y2-Ep1oAvXSSf5rJ6l5BYbp-Ssj1CQVaQJ6C1NvQJUwOdVmyXnPh0aGb6CYvsCbP-17qCzBtNn6yRR54FcbH7Nukv5qPL3aWb_PNOi1pCFps=86B174BE")

# Store last seen issues
last_seen_issues = {}

def fetch_jira_issues():
    """Fetch issues from Jira API."""
    print(JIRA_API_ENDPOINT)
    response = requests.get(JIRA_API_ENDPOINT, headers=HEADERS, auth=AUTH)
    print("Status Code:", response.status_code)
    print("Response Body:", response.text)  # Debugging output

    if response.status_code == 200:
        return response.json().get("issues", [])
    else:
        print(f"Error fetching Jira issues: {response.text}")
        return []

def format_timestamp(timestamp):
    """Convert Jira timestamp to a readable format."""
    try:
        dt = datetime.strptime(timestamp, "%Y-%m-%dT%H:%M:%S.%f%z")
        return dt.strftime("%Y-%m-%d %H:%M:%S")  # Format: YYYY-MM-DD HH:MM:SS
    except Exception as e:
        print(f"Error formatting timestamp: {e}")
        return timestamp  # Return original if parsing fails

def send_slack_message(issue):
    """Send a Slack notification with timestamp."""
    slack_url = "https://slack.com/api/chat.postMessage"
    headers = {
        "Authorization": f"Bearer {SLACK_TOKEN}",
        "Content-Type": "application/json"
    }
    
    issue_key = issue['key']
    summary = issue['fields']['summary']
    status = issue['fields']['status']['name']
    last_updated = format_timestamp(issue['fields']['updated'])  # Convert timestamp

    message = (
        f" *Jira Update*\n"
        f" *{issue_key}* - {summary}\n"
        f" Status: {status}\n"
        f" Last Updated: {last_updated}\n"
        f" <{JIRA_URL}/browse/{issue_key}|View in Jira>"
    )

    data = {
        "channel": SLACK_USER_ID,  # Slack User ID
        "text": message
    }
    
    response = requests.post(slack_url, headers=headers, json=data)
    if response.status_code == 200 and response.json().get("ok"):
        print(f" Slack message sent for {issue_key}")
    else:
        print(f" Failed to send Slack message: {response.text}")

def track_issues():
    """Poll Jira for new/updated issues and send Slack notifications."""
    global last_seen_issues
    while True:
        issues = fetch_jira_issues()
        for issue in issues:
            issue_key = issue['key']
            summary = issue['fields']['summary']
            status = issue['fields']['status']['name']
            last_updated = issue['fields']['updated']

            print(f"Ticket Key: {issue_key}")
            print(f"Summary: {summary}")
            print(f"Status: {status}")
            print(f"Last Updated: {format_timestamp(last_updated)}")
            print("-" * 40)

            # If issue is new or updated, send notification
            if issue_key not in last_seen_issues or last_seen_issues[issue_key] != last_updated:
                send_slack_message(issue)

            # Update last seen status
            last_seen_issues[issue_key] = last_updated

        time.sleep(60)  # Wait 60 seconds before checking again

if __name__ == "__main__":
    print(" Jira-Slack bot is running...")
    track_issues()

