
# ## Challenge Review: Analyzing App.py & Axel Binary

### ### 1. Initial Analysis of the Server

After analyzing the `app.py` and the provided `Dockerfile`, I focused on the main functions of the server to understand the application flow. The most interesting part was the `run_axel_download()` function. This function uses a binary of a program named `/usr/local/bin/axel` to download files that the user sends to it.

I realized that the binary is based on the open-source project:

> [Axel Download Accelerator](https://github.com/axel-download-accelerator/axel)

### ### 2. Deep Dive into the Binary

I dived into the repository and studied the manual of the binary found here: [Axel Documentation](https://github.com/axel-download-accelerator/axel/blob/master/doc/axel.txt). It is a normal binary that downloads files from a URL into the server.

**My Findings on Command Injection:** The function takes one argument (`url`) and runs the binary with the user-given argument as a parameter. However, after checking the implementation, the shell is **not** set to `True`. This means there is no shell parsing in this process, which makes a direct **Command Injection impossible**.

### ### 3. Identifying the Vulnerability (SSRF & Path Traversal)

Since the app downloads a given URL, it was possible to perform **SSRF** to download local server files into `list_downloads()`. However, a more direct path was through the `download()` function.

The `download()` function takes a `filename` from the user and joins it with the `FILE_DIRECTORY` (which is `/app`). Then, it attempts to serve the file via `send_file()`.

Python

```
target = FILES_DIR / filename
```

**The Exploitation Logic:** I decided to try a **Path Traversal** attack. If we send a filename like `/../../../../flag.txt`, the path joins as follows: `target = FILES_DIR / filename` --> `/app/../../../../flag.txt` which resolves to simply **`/flag.txt`**.

I actually found a similar issue explained on GitHub before I even finalized my exploit, which confirmed my theory: [GitHub Issue #21](https://github.com/RipudamanKaushikDal/projects/issues/21).

### ### 4. Server Routes & Attack Vector

- `GET /`: Main endpoint that just lists all downloads by executing `list_downloads()`.
    
- `POST /fetch`: Executes `run_axel_download(url)` and then `list_downloads()`.
    
- `GET /download/<path:filename>`: Downloads a specific file from the server.
    

**The Strategy:** I realized we don't even need to request the file through `/fetch` first and then download it. We can exploit the Path Traversal to download the `flag.txt` directly. To prevent the browser from automatically "fixing" or normalizing the URL (which would remove the `../`), I used `curl` with the `--path-as-is` flag.

### ### 5. Final Execution

By sending the following request, I bypassed the directory restriction and retrieved the flag:

Bash

```
curl --path-as-is "http://46.225.117.62:30020/download/../../../../flag.txt"
```

**Flag Captured:** `upCTF{4x3l_0d4y_w1th4_tw1st-XkMH49XCaeeb50d6}`