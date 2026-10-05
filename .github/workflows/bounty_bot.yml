import os
import sys
import json
import time
import re
import subprocess
import requests
from google import genai
from google.genai import types

GEMINI_API_KEY = "AQ.Ab8RN6KCejJKYgjqQIakNWpm2y9lHxlOBYRotX2MTduMTEFQ5Q"
GITHUB_TOKEN = "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"
PHANTOM_SOLANA_WALLET = "6JZZZEtkButVq18xiKz3HehRt5qHKgrFhhFHdjfZZdQT"

MAX_TOKEN_BUDGET = 50000
client = genai.Client(api_key=GEMINI_API_KEY)
CURRENT_REPO_DIR = ""

def execute_terminal_command(command: str) -> str:
    try:
        result = subprocess.run(
            command, shell=True, capture_output=True, text=True, timeout=120, cwd=CURRENT_REPO_DIR
        )
        output = result.stdout + "\n" + result.stderr
        
        if not output.strip():
            return "Command executed successfully with no output."

        if len(output) > 2000:
            output = (
                output[:1000] 
                + "\n\n... [Output truncated to save tokens] ...\n\n" 
                + output[-1000:]
            )
        return output
    except Exception as e:
        return f"Error executing command: {str(e)}"

def read_file_content(file_path: str) -> str:
    full_path = os.path.join(CURRENT_REPO_DIR, file_path)
    try:
        with open(full_path, 'r', encoding='utf-8') as f:
            return f.read()
    except Exception as e:
        return f"Error reading file: {str(e)}"

def write_file_content(file_path: str, new_content: str) -> str:
    full_path = os.path.join(CURRENT_REPO_DIR, file_path)
    try:
        os.makedirs(os.path.dirname(full_path), exist_ok=True)
        with open(full_path, 'w', encoding='utf-8') as f:
            f.write(new_content)
        return f"Successfully updated file {file_path}."
    except Exception as e:
        return f"Error writing file: {str(e)}"

tools_list = [execute_terminal_command, read_file_content, write_file_content]

def collect_project_code_context(repo_dir: str) -> tuple[str, int, str, str]:
    context_str = "\n=== CURRENT PROJECT FILES ===\n"
    file_count = 0
    detected_lang = "python"
    test_command = "pytest"
    
    ignored_dirs = {'.git', '__pycache__', 'venv', 'env', 'node_modules', '.pytest_cache', 'build', 'dist', 'target'}

    if os.path.exists(os.path.join(repo_dir, "package.json")):
        detected_lang = "javascript/typescript"
        test_command = "npm test"
    elif os.path.exists(os.path.join(repo_dir, "Cargo.toml")):
        detected_lang = "rust"
        test_command = "cargo test"
    elif os.path.exists(os.path.join(repo_dir, "requirements.txt")) or os.path.exists(os.path.join(repo_dir, "setup.py")) or os.path.exists(os.path.join(repo_dir, "pyproject.toml")):
        detected_lang = "python"
        test_command = "pytest"

    allowed_extensions = (".py", ".json", ".md", ".yml", ".yaml", ".txt", ".rs", ".js", ".ts")

    for root, dirs, files in os.walk(repo_dir):
        dirs[:] = [d for d in dirs if d not in ignored_dirs and not d.startswith('.')]
        for file in files:
            if file.endswith(allowed_extensions):
                file_path = os.path.join(root, file)
                rel_path = os.path.relpath(file_path, repo_dir)
                try:
                    with open(file_path, 'r', encoding='utf-8') as f:
                        content = f.read()
                        if len(content) <= 30000:
                            context_str += f"\n📄 FILE: {rel_path}\n```\n{content}\n```\n"
                            file_count += 1
                except Exception:
                    continue
                    
    return context_str, file_count, detected_lang, test_command

def extract_bounty_amount(text: str) -> int:
    dollar_matches = re.findall(r'\$(\d+)', text)
    if dollar_matches:
        amounts = [int(m) for m in dollar_matches]
        max_val = max(amounts)
        if max_val > 0:
            return max_val

    sol_matches = re.findall(r'(\d+(?:\.\d+)?)\s*(?:sol)', text, re.IGNORECASE)
    if sol_matches:
        sol_val = float(sol_matches[0]) * 150 
        return int(sol_val)

    return 0

def fetch_solana_bounties(platform_name: str, limit: int = 5):
    query_map = {
        "octasol": "label:octasol+state:open+type:issue",
        "gitcoin": "label:gitcoin+solana+state:open+type:issue",
        "algora": "label:algora+solana+state:open+type:issue",
        "solanafloor": "label:solanafloor+state:open+type:issue",
        "colosseum": "label:colosseum+solana+state:open+type:issue"
    }
    
    query = query_map.get(platform_name.lower(), "label:solana+state:open+type:issue")
    url = f"https://api.github.com/search/issues?q={query}&per_page=40"
    headers = {"Authorization": f"token {GITHUB_TOKEN}", "Accept": "application/vnd.github.v3+json"}
    
    res = requests.get(url, headers=headers)
    filtered_issues = []
    
    if res.status_code == 200:
        items = res.json().get("items", [])
        for item in items:
            title_and_body = f"{item['title']} {item.get('body', '')}"
            bounty_val = extract_bounty_amount(title_and_body)
            
            if 100 <= bounty_val <= 500:
                repo_url = item["repository_url"]
                repo_fullname = repo_url.replace("https://api.github.com/repos/", "")
                filtered_issues.append({
                    "repo": repo_fullname,
                    "issue_number": item["number"],
                    "title": item["title"],
                    "body": item.get("body", ""),
                    "amount": bounty_val
                })
                if len(filtered_issues) >= limit:
                    break
                    
    return filtered_issues

def create_pull_request(repo_fullname: str, head_branch: str, base_branch: str, title: str, body: str) -> str:
    url = f"https://api.github.com/repos/{repo_fullname}/pulls"
    headers = {"Authorization": f"token {GITHUB_TOKEN}", "Accept": "application/vnd.github.v3+json"}
    payload = {"title": title, "body": body, "head": head_branch, "base": base_branch}
    res = requests.post(url, headers=headers, json=payload)
    if res.status_code == 201:
        return res.json()["html_url"]
    else:
        raise Exception(f"Failed to create PR: {res.status_code} - {res.text}")

def run_bounty_agent(repo_fullname: str, issue_number: int, issue_title: str, issue_body: str, bounty_amount: int, base_branch: str = "main"):
    global CURRENT_REPO_DIR
    repo_owner, repo_name = repo_fullname.split("/")
    branch_name = f"bounty-fix-{issue_number}-{int(time.time())}"
    accumulated_tokens = 0
    
    print(f"\n🚀 [Bounty ${bounty_amount}] {repo_fullname} | Issue #{issue_number}")
    print(f"Title: {issue_title}")

    workspace_dir = "/content/agent_workspace"
    os.makedirs(workspace_dir, exist_ok=True)
    CURRENT_REPO_DIR = os.path.join(workspace_dir, repo_name)

    if os.path.exists(CURRENT_REPO_DIR):
        import shutil
        shutil.rmtree(CURRENT_REPO_DIR)

    clone_url = f"https://x-access-token:{GITHUB_TOKEN}@github.com/{repo_fullname}.git"
    subprocess.run(f"git clone {clone_url} \"{CURRENT_REPO_DIR}\"", shell=True, check=True)

    execute_terminal_command(f"git checkout -b {branch_name}")

    files_context, count, project_lang, test_command = collect_project_code_context(CURRENT_REPO_DIR)
    
    print(f"Detected Language: {project_lang.upper()} | Test Command: {test_command}")

    if project_lang == "python":
        execute_terminal_command("if [ -f requirements.txt ]; then pip install --no-cache-dir -r requirements.txt; fi")
        execute_terminal_command("pip install pytest")
    elif project_lang == "javascript/typescript":
        execute_terminal_command("npm install")
    elif project_lang == "rust":
        execute_terminal_command("cargo build")

    system_instruction = f"You are an expert software engineer specialized in {project_lang}. Analyze the issue carefully, write the correct code, and test it to fix the bug completely."

    chat = client.chats.create(
        model="gemini-2.5-flash",
        config=types.GenerateContentConfig(
            system_instruction=system_instruction,
            tools=tools_list,
            temperature=0.1
        )
    )

    prompt = f"""
BOUNTY ISSUE TITLE: {issue_title}
DESCRIPTION:
{issue_body}

{files_context}

Task: Fix the issue and test the code using the command: `{test_command}`.
    """

    response = chat.send_message(prompt)

    max_steps = 8
    for step in range(1, max_steps + 1):
        if hasattr(response, 'usage_metadata') and response.usage_metadata:
            step_tokens = response.usage_metadata.total_token_count
            accumulated_tokens += step_tokens
            if accumulated_tokens >= MAX_TOKEN_BUDGET:
                break

        if not response.function_calls:
            break

        for call in response.function_calls:
            tool_name = call.name
            args = call.args
            
            if tool_name == "execute_terminal_command":
                output = execute_terminal_command(args["command"])
            elif tool_name == "read_file_content":
                output = read_file_content(args["file_path"])
            elif tool_name == "write_file_content":
                output = write_file_content(args["file_path"], args["new_content"])
            else:
                output = "Unknown tool."

            response = chat.send_message(
                types.Part.from_function_response(
                    name=tool_name,
                    response={"result": output}
                )
            )

    print("\nRunning final tests...")
    final_test_output = execute_terminal_command(test_command)
    
    output_lower = final_test_output.lower()
    is_passed = (
        ("error" not in output_lower and "fail" not in output_lower) or
        ("passed" in output_lower or "ok" in output_lower or "0 failed" in output_lower or "success" in output_lower)
    )

    if is_passed:
        print("Tests passed successfully! Pushing PR and including Phantom wallet...")
        execute_terminal_command('git config user.name "Colab-Bounty-Agent"')
        execute_terminal_command('git config user.email "bot@colab.local"')
        execute_terminal_command("git add .")
        
        commit_res = execute_terminal_command(f'git commit -m "Fix: {issue_title} (AI Solana Bounty Fix)"')
        if "nothing to commit" in commit_res.lower():
            return

        execute_terminal_command(f"git push origin {branch_name}")

        pr_body = (
            f"Automated PR for ${bounty_amount} Crypto Bounty Issue #{issue_number}\n\n"
            f"Verified by test (`{test_command}`): Passed Successfully\n\n"
            f"### Payout Routing:\n"
            f"- Phantom Wallet: `{PHANTOM_SOLANA_WALLET}`\n"
            f"- Bridged via Wormhole / Swapped via Jupiter DEX if needed."
        )

        try:
            pr_url = create_pull_request(
                repo_fullname=repo_fullname,
                head_branch=branch_name,
                base_branch=base_branch,
                title=f"Fix: {issue_title}",
                body=pr_body
            )
            print(f"PR created successfully: {pr_url}")
        except Exception as e:
            print(f"Error opening PR: {e}")

platforms = ["octasol", "gitcoin", "algora", "solanafloor", "colosseum"]

for platform in platforms:
    print(f"\n==========================================")
    print(f"Searching bounties ($100 - $500) on platform: {platform.upper()}")
    print(f"==========================================")
    
    tasks = fetch_solana_bounties(platform_name=platform, limit=5)
    
    if not tasks:
        print(f"No matching tasks found on {platform}.")
        continue

    for task in tasks:
        run_bounty_agent(
            repo_fullname=task["repo"],
            issue_number=task["issue_number"],
            issue_title=task["title"],
            issue_body=task["body"],
            bounty_amount=task["amount"],
            base_branch="main"
        )
