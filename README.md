অবশ্যই, আপনার V2Ray/Xray লগ ফাইল থেকে কানেকশন তথ্য বিশ্লেষণ করার জন্য সম্পূর্ণ Python স্ক্রিপ্টটি নিচে দেওয়া হলো। এটি গত N মিনিটের মধ্যে কানেক্টেড ইউজার এবং তাদের আইপি অ্যাড্রেসগুলো দেখাবে।

এই স্ক্রিপ্টটি ব্যবহার করার জন্য:

1.  **স্ক্রিপ্টটি সেভ করুন:** কোডটি `v2ray_monitor.py` নামে একটি ফাইলে সেভ করুন।
2.  **পাথ যাচাই করুন:** স্ক্রিপ্টের শুরুতে `LOG_FILE_PATH` ভেরিয়েবলে আপনার V2Ray/Xray অ্যাক্সেস লগ ফাইলের সঠিক পাথ সেট করা আছে কিনা, তা নিশ্চিত করুন। সাধারণত এটি `/var/log/v2ray/access.log` অথবা `/var/log/xray/access.log` হয়।
3.  **রান করুন:** আপনার সার্ভারের টার্মিনালে এই কমান্ডটি চালান:
    ```bash
    python3 v2ray_monitor.py
    ```

### **Python স্ক্রিপ্ট (`v2ray_monitor.py`):**

```python
import json
import datetime
import subprocess
import argparse
from collections import defaultdict

# --- কনফিগারেশন (প্রয়োজনে পরিবর্তন করুন) ---
# আপনার V2Ray/Xray লগ ফাইলের ডিফল্ট পাথ।
# যদি এটি ভিন্ন হয়, তবে স্ক্রিপ্ট চালানোর সময় --log আর্গুমেন্ট ব্যবহার করতে পারেন।
LOG_FILE_PATH = "/var/log/v2ray/access.log" # Xray হলে সাধারণত /var/log/xray/access.log হয়

def parse_log_entry(line):
    """
    লগ লাইন থেকে JSON এন্ট্রি পার্স করে।
    যদি পার্স করতে সমস্যা হয়, None রিটার্ন করে।
    """
    try:
        return json.loads(line)
    except json.JSONDecodeError:
        return None

def analyze_v2ray_logs(log_lines, history_minutes):
    """
    লগ লাইনগুলো বিশ্লেষণ করে "বর্তমান" (শেষ কিছু লগ এন্ট্রি)
    এবং গত N মিনিটের মধ্যে কানেক্টেড ইউজার ও তাদের আইপি অ্যাড্রেসগুলো বের করে।

    V2Ray সরাসরি "সক্রিয়" কানেকশন দেখায় না, এটি শুধুমাত্র কানেকশনের রেকর্ড রাখে।
    তাই "বর্তমান" বলতে আমরা লগ ফাইলের শেষ কিছু কানেকশন এন্ট্রিকে ধরে নিচ্ছি।
    """
    total_current_ips = set() # বর্তমানের মোট ইউনিক আইপি
    user_current_ips = defaultdict(set) # ইউজার-ভিত্তিক বর্তমান ইউনিক আইপি
    user_history_ips = defaultdict(set) # ইউজার-ভিত্তিক ঐতিহাসিক (গত N মিনিটে) ইউনিক আইপি

    # বর্তমান UTC সময় (V2Ray লগ UTC টাইমস্ট্যাম্প ব্যবহার করে)
    now = datetime.datetime.now(datetime.timezone.utc)
    history_cutoff_time = now - datetime.timedelta(minutes=history_minutes)

    for line in log_lines:
        entry = parse_log_entry(line)
        if not entry:
            continue

        # লগ এন্ট্রি থেকে প্রয়োজনীয় তথ্য বের করা
        user_data = entry.get('user')
        if not user_data or not isinstance(user_data, dict):
            continue

        user_id = user_data.get('id') # ইউজার আইডি (যেমন: UUID)
        ip_address = entry.get('ip') # সোর্স আইপি অ্যাড্রেস
        timestamp_str = entry.get('time') # কানেকশনের টাইমস্ট্যাম্প (যেমন: "2023-10-27T10:00:00.123Z")

        if not user_id or not ip_address or not timestamp_str:
            continue

        try:
            # ISO ফরম্যাট টাইমস্ট্যাম্প পার্স করা ('Z' মানে UTC, তাই '+00:00' দিয়ে প্রতিস্থাপন)
            entry_time = datetime.datetime.fromisoformat(timestamp_str.replace('Z', '+00:00'))
        except ValueError:
            # যদি টাইমস্ট্যাম্প ফরম্যাট ভুল হয়, এই এন্ট্রি বাদ দাও
            continue

        # "বর্তমান" কানেকশনগুলি ট্র্যাক করা (লগ ফাইলের শেষ এন্ট্রিগুলো থেকে)
        total_current_ips.add(ip_address)
        user_current_ips[user_id].add(ip_address)

        # গত N মিনিটের মধ্যে কানেক্টেড আইপি ট্র্যাক করা
        if entry_time >= history_cutoff_time:
            user_history_ips[user_id].add(ip_address)

    return total_current_ips, user_current_ips, user_history_ips

def main():
    """
    স্ক্রিপ্টের প্রধান ফাংশন, যা আর্গুমেন্ট পার্স করে, লগ পড়ে এবং ফলাফল প্রিন্ট করে।
    """
    parser = argparse.ArgumentParser(
        description="V2Ray/Xray connection monitor using access logs.",
        formatter_class=argparse.RawTextHelpFormatter # ফর্ম্যাটিং ঠিক রাখার জন্য
    )
    parser.add_argument(
        "--log",
        default=LOG_FILE_PATH,
        help=(
            f"Path to V2Ray/Xray access log file.\n"
            f"Default: {LOG_FILE_PATH}"
        )
    )
    parser.add_argument(
        "--tail",
        type=int,
        default=1000,
        help=(
            "Number of last lines to read from the log file for 'current' connections.\n"
            "A higher number provides a broader view of recent activity.\n"
            "Default: 1000"
        )
    )
    parser.add_argument(
        "--history",
        type=int,
        default=20,
        help=(
            "Time window in minutes for 'historical' connections (e.g., 10 for last 10 mins).\n"
            "Default: 20"
        )
    )
    
    args = parser.parse_args()

    print(f"--- Analyzing V2Ray/Xray access log: {args.log} ---")
    print(f"Reading last {args.tail} log entries for current view.")
    print(f"Checking connections from the last {args.history} minutes for historical view.")

    # tail কমান্ড ব্যবহার করে লগ ফাইলের শেষ N লাইন পড়া
    try:
        # subprocess.run একটি কমান্ড এক্সিকিউট করে
        result = subprocess.run(
            ['tail', '-n', str(args.tail), args.log],
            capture_output=True, # আউটপুট ক্যাপচার করে
            text=True,           # আউটপুটকে টেক্সট হিসাবে ডিকোড করে
            check=True           # যদি কমান্ড ভুল করে, তাহলে CalledProcessError দেয়
        )
        log_lines = result.stdout.strip().split('\n')
        # যদি লগ ফাইল খালি থাকে, তাহলে split('\n') একটি [''] লিস্ট দেবে, যা ফিল্টার করা উচিত
        if len(log_lines) == 1 and log_lines[0] == '':
            log_lines = [] # খালি করে দাও
            
    except FileNotFoundError:
        print(f"\nError: Log file not found at '{args.log}'")
        print("Please check the log file path. Make sure V2Ray/Xray is configured to write access logs.")
        return
    except subprocess.CalledProcessError as e:
        print(f"\nError running 'tail' command: {e}")
        print(f"Command stderr: {e.stderr.strip()}")
        print("Ensure 'tail' command is available and you have read permissions for the log file.")
        return
    except Exception as e:
        print(f"\nAn unexpected error occurred: {e}")
        return

    if not log_lines:
        print("\nNo log entries found. The log file might be empty or the path is incorrect.")
        return

    # লগ ডেটা বিশ্লেষণ করা
    total_current_ips, user_current_ips, user_history_ips = analyze_v2ray_logs(log_lines, args.history)

    # --- ফলাফল প্রিন্ট করা ---
    print("\n--- Current Connections (from last {} log entries) ---".format(args.tail))
    print(f"Total unique IPs connected: {len(total_current_ips)}")
    
    print("\nUser-wise current connections:")
    if not user_current_ips:
        print("  No recent connections found in the last {} log entries.".format(args.tail))
    else:
        # ইউজার আইডি অনুযায়ী সাজিয়ে প্রিন্ট করা
        for user_id, ips in sorted(user_current_ips.items()):
            print(f"  User '{user_id}': {len(ips)} unique IP(s) - {', '.join(sorted(list(ips)))}")

    print(f"\n--- Historical Connections (last {args.history} minutes) ---")
    print("\nUser-wise historical connections:")
    if not user_history_ips:
        print(f"  No connections found in the last {args.history} minutes.")
    else:
        # ইউজার আইডি অনুযায়ী সাজিয়ে প্রিন্ট করা
        for user_id, ips in sorted(user_history_ips.items()):
            print(f"  User '{user_id}': {len(ips)} unique IP(s) - {', '.join(sorted(list(ips)))}")

if __name__ == "__main__":
    main()
```
