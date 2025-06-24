## What is this fork?
This fork is a patch of the original, replacing the use of httpx with `asyncio requests`
This is mainly for my personal projects.

## Why was this made? 
I have encountered issues where httpx is simply incapable of requesting data from `roblox.com`
I belive this to be a issue in httpx itself, as i can use `curl` to get data from `roblox.com` fine.

Here's a test script to tell you if your `httpx` is capable of reading data from `roblox.com`
If it's incapable, it will timeout, because the server will not return ANY communication after doing a handshake and negitiating about `http1 http2xx`

```py
import httpx

def fetch_roblox_user_data(user_id: int):
    """
    Fetches user data from Roblox API using httpx and prints it.
    Handles potential HTTP and request errors.

    Args:
        user_id (int): The Roblox user ID to fetch data for.
    """
    url = f"https://users.roblox.com/v1/users/{user_id}"
    print(f"Attempting to fetch data from: {url}")

    try:
        # Send a GET request to the specified URL
        response = httpx.get(url, timeout=10.0) # Added a timeout for robustness

        # Raise an exception for HTTP errors (4xx or 5xx)
        response.raise_for_status()

        # If the request was successful, print the JSON response
        print("\nSuccessfully fetched data:")
        print(response.json())

    except httpx.HTTPError as e:
        # Handle HTTP-related errors (e.g., HTTP status errors, connection errors)
        print(f"\nAn HTTPX error occurred: {e}")
        if e.request:
            print(f"Request URL: {e.request.url}")
        if hasattr(e, 'response') and e.response:
            print(f"Response Status: {e.response.status_code}")
            print(f"Response Text: {e.response.text}")
    except Exception as e:
        # Catch any other unexpected errors
        print(f"\nAn unexpected error occurred: {e}")

if __name__ == "__main__":
    roblox_user_id = 949472658
    fetch_roblox_user_data(roblox_user_id)
```



**I belive httpx has a issue in negotiation**


A more advanced script trying different headers, keepalive, and SSL, is here:


```py
import time
import httpx
import json
import subprocess
import os

def run_curl_command(url, headers):
    """
    Runs a curl command to check the API status.
    Returns True if curl succeeds (HTTP 2xx), False otherwise.
    """
    header_str = " ".join([f"-H '{k}: {v}'" for k, v in headers.items()])
    command = f"curl -k -v --compressed -X GET {header_str} '{url}'"
    
    print(f"\n--- Running curl command to verify API status ---")
    print(f"Executing: {command}")
    
    try:
        result = subprocess.run(
            command, 
            shell=True, 
            capture_output=True, 
            text=True, 
            timeout=10, 
            check=False
        )
        
        print("Curl stdout:\n", result.stdout)
        print("Curl stderr:\n", result.stderr)

        if "HTTP/" in result.stderr:
            for line in result.stderr.splitlines():
                if line.startswith("< HTTP/") and (" 200 " in line or " 204 " in line):
                    print("Curl verification: API is UP and accessible via curl (HTTP 2xx detected).")
                    return True
                elif line.startswith("< HTTP/") and (" 4" in line or " 5" in line):
                    print(f"Curl verification: API returned an error status: {line}")
                    return False
        
        print("Curl verification: Could not confirm successful HTTP status from curl output.")
        return False
        
    except FileNotFoundError:
        print("Curl command not found. Please ensure curl is installed and in your PATH.")
        return False
    except subprocess.TimeoutExpired:
        print("Curl command timed out.")
        return False
    except Exception as e:
        print(f"Error running curl command: {e}")
        return False

def check_roblox_api_final_attempt():
    retries = 2 # Reduced retries for faster feedback
    url = "https://users.roblox.com/v1/users/949472658"
    
    # Headers that curl uses, which work for curl.
    # Let's try matching curl's `Accept` and minimal set.
    headers_curl_mimic = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36",
        "Accept": "*/*", # Curl's common default
        "Accept-Encoding": "gzip, deflate, br", # Keep what curl had if it used it.
        "Accept-Language": "en-US,en;q=0.9",
        # "Connection: close" is implicitly handled by HTTP/2 and should not be explicitly set.
    }

    api_is_up = run_curl_command(url, headers_curl_mimic) # Use the same headers for curl test
    if not api_is_up:
        print("Curl verification failed or API is down. Aborting httpx attempts.")
        return
    else:
        print("Proceeding with httpx attempts...")

    # --- HTTPX Attempts with HTTP/2 and final header set ---
    
    # Final Trial: Curl-mimicking headers, http2=True, lower timeout, verify=True/False
    print(f"\n--- HTTPX Final Trial: Curl-Mimicking Headers, http2=True, timeout=15 ---")
    
    # Test with verify=True first (recommended)
    for attempt in range(1, retries + 1):
        try:
            print(f"Attempt {attempt}: Trying with curl-mimic headers and verify=True...")
            with httpx.Client(timeout=15, http2=True, verify=True) as client: # Lowered timeout
                resp = client.get(url, headers=headers_curl_mimic)
                resp.raise_for_status()
                print("Status code:", resp.status_code)
                print("Response:", resp.json())
                return
        except httpx.HTTPStatusError as e:
            print(f"Attempt {attempt}: HTTP error: {e.response.status_code} - {e.response.text}")
        except httpx.RequestError as e:
            print(f"Attempt {attempt}: Request error: {type(e).__name__}: {e}")
        except Exception as e:
            print(f"Attempt {attempt}: Unexpected error: {type(e).__name__}: {e}")
        time.sleep(1) # Shorter sleep
    
    # Then with verify=False as a last resort
    for attempt in range(1, retries + 1):
        try:
            print(f"Attempt {attempt}: Trying with curl-mimic headers and verify=False...")
            with httpx.Client(timeout=15, http2=True, verify=False) as client: # Lowered timeout
                resp = client.get(url, headers=headers_curl_mimic)
                resp.raise_for_status()
                print("Status code:", resp.status_code)
                print("Response:", resp.json())
                return
        except httpx.HTTPStatusError as e:
            print(f"Attempt {attempt}: HTTP error: {e.response.status_code} - {e.response.text}")
        except httpx.RequestError as e:
            print(f"Attempt {attempt}: Request error: {type(e).__name__}: {e}")
        except Exception as e:
            print(f"Attempt {attempt}: Unexpected error: {type(e).__name__}: {e}")
        time.sleep(1) # Shorter sleep


    print("Failed to contact Roblox API after several httpx attempts.")

if __name__ == "__main__":
    check_roblox_api_final_attempt()
```


## END OF CHANGES. FROM HERE ON IT'S THE NORMAL README.
### If you need to use this, you're a poor soul. 


<img align="left" src="./gh-assets/logo-wordmark.svg" alt="ro.py" height="96" /><a href="https://ro.py.jmk.gg"><img align="right" src="./gh-assets/docs-button.svg" alt="Docs"></a><a href="https://discord.gg/hHjwxZxhR2"><img align="right" src="./gh-assets/discord-button.svg" alt="Discord"></a><img src="./gh-assets/clearfloat.svg">

## Overview
ro.py is an asynchronous, object-oriented wrapper for the Roblox web API.

## Features
- **Asynchronous**: ro.py works well with asynchronous frameworks like [FastAPI](https://fastapi.tiangolo.com/) and 
[discord.py](https://github.com/Rapptz/discord.py).  
- **Easy**: ro.py's client-based model is intuitive and easy to learn. 
  It abstracts away API requests and leaves you with simple objects that represent data on the Roblox platform.
- **Flexible**: ro.py's Requests object allows you to extend ro.py beyond what we've already implemented.

## Installation
To install the latest stable version of ro.py, run the following command:
```
python3 -m pip install roblox
```

To install the latest **unstable** version of ro.py, install [git-scm](https://git-scm.com/downloads) and run the following:
```
python3 -m pip install git+https://github.com/ro-py/ro.py.git
```

## Support
- Learn how to use ro.py in the docs: https://ro.py.jmk.gg/dev/tutorials/  
- The [RoAPI Discord server](https://discord.gg/a69neqaNZ5) provides support for ro.py in the `#ro.py-support` channel.
