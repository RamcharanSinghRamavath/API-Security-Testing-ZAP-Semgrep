# Local API Security Testing Lab
*Step-by-step setup, OWASP ZAP API scan, Semgrep SAST, fixes, evidence, and troubleshooting*

This guide records the local practice lab and its commands. Follow it only on systems you own or have permission to test.

## 1. Purpose and scope

This guide documents the local practice lab built during this conversation: a small Python/FastAPI API, OWASP ZAP API scanning using an OpenAPI document, and Semgrep static code analysis. It is designed to be repeated on a personal Kali Linux laptop.

Use this only on your own local practice API or systems for which you have explicit permission. Do not scan company or third-party systems without authorization. Keep secrets, tokens, internal URLs, and sensitive source code out of shared reports.

Important limitation: a scan with zero findings does not prove that an application is secure. It only means the enabled rules/tests did not report an issue within the tested scope.


## 2. What each tool does

- FastAPI: creates the small practice API in Python.
- Uvicorn: runs the FastAPI app locally.
- OpenAPI JSON: machine-readable description of API endpoints; FastAPI serves it at /openapi.json by default.
- Docker: runs OWASP ZAP in a container, avoiding a separate ZAP installation.
- OWASP ZAP API scan (DAST): sends test requests to the running API using its OpenAPI specification.
- Semgrep (SAST): inspects source code for patterns that may indicate security issues.
- curl: checks HTTP responses and response headers from the local API.
- nano: terminal text editor used to create and edit Python files.

## 3. Prerequisites and verify tools

The commands below assume Kali Linux (or a Debian-based Linux environment), a terminal, Python 3, and internet access for package/image downloads. On a personal laptop, keep the API bound to 127.0.0.1 so it is accessible only locally.

```bash
cat /etc/os-release
python3 --version
python3 -m pip --version
git --version
curl --version
docker --version
semgrep --version
```

What these checks mean: they print the OS and versions so you can identify missing tools before proceeding. In this lab, Docker was available and `sudo docker run --rm hello-world` succeeded. Semgrep was installed using pipx and its version command worked.

### If a command is missing

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip pipx curl git
```

These apt commands install common prerequisites on Kali/Debian. If Docker is missing, follow the current Docker Engine installation instructions for your exact OS rather than assuming a package name or repository is already configured. After Docker installation, verify with `docker --version` and `sudo docker run --rm hello-world`.

### Install Semgrep with pipx if it is not installed

```bash
pipx ensurepath
pipx install semgrep
```

If pipx says Semgrep is already installed, do not reinstall it unnecessarily. Open a new terminal if PATH changes do not take effect, then run `semgrep --version`.


## 4. Create the practice API

Create a project directory and a Python virtual environment. The virtual environment keeps this lab's Python packages separate from system Python.

```bash
mkdir -p ~/api-lab
cd ~/api-lab
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install fastapi uvicorn
```

The terminal prompt should usually show `(.venv)` while the environment is active. If you open a new terminal later, activate it again with `cd ~/api-lab && source .venv/bin/activate`.

### Create app.py

```bash
nano app.py
```

Paste the following initial API code into nano:

```bash
from fastapi import FastAPI

app = FastAPI(title="Cybersecurity Practice API", version="1.0.0")

@app.get("/")
def home():
    return {"message": "Local practice API is running"}

@app.get("/hello/{name}")
def hello(name: str):
    return {"message": f"Hello, {name}"}
```

Save in nano: press Ctrl+O, press Enter, then Ctrl+X. This defines two GET endpoints: `/` and `/hello/{name}`.

### Run the API locally

```bash
cd ~/api-lab
source .venv/bin/activate
uvicorn app:app --host 127.0.0.1 --port 8000
```

Leave this terminal running. `127.0.0.1` means the service listens on the local machine only. Avoid `--host 0.0.0.0` for this exercise unless you specifically need other devices to reach it and understand the exposure.

### Verify in a second terminal

```bash
curl -i http://127.0.0.1:8000/
curl -i http://127.0.0.1:8000/hello/Ramcharan
curl -s http://127.0.0.1:8000/openapi.json
```

Expected: the first endpoint returns JSON containing `Local practice API is running`; the second returns a greeting; `/openapi.json` returns the API definition. You can also open `http://127.0.0.1:8000/` and `http://127.0.0.1:8000/docs` in a browser.


## 5. First OWASP ZAP API scan

The lab used the official ZAP stable Docker image. The API must be running while the scan runs. Keep the API terminal open.

```bash
mkdir -p ~/zap-reports
sudo docker run --rm \\
  --network host \\
  -v "$HOME/zap-reports:/zap/wrk/:rw" \\
  ghcr.io/zaproxy/zaproxy:stable \\
  zap-api-scan.py \\
  -t http://127.0.0.1:8000/openapi.json \\
  -f openapi \\
  -r /zap/wrk/zap-report.html
```

Command breakdown: `docker run --rm` runs a temporary container and removes it afterwards; `--network host` lets the container reach the API on the host's loopback interface in this Linux lab; `-v` mounts the reports directory; `-t` specifies the OpenAPI URL; `-f openapi` tells ZAP the target is an OpenAPI definition; `-r` writes the HTML report.

In the first scan, the output reported 0 FAIL alerts and 2 warning types: missing X-Content-Type-Options [10021] and missing/invalid Cross-Origin-Resource-Policy [90004]. The scan also showed many PASS checks. These results apply to this local practice API and that scan only.

### Open the report

```bash
xdg-open ~/zap-reports/zap-report.html
```

If the report file has a different name, list the directory first: `ls -lh ~/zap-reports`. In a later command, the user opened `~/zap-reports/zap-report-final.html`; that filename is only correct if that file actually exists.


## 6. Add security response headers

The first ZAP scan identified missing headers. We added middleware to app.py. Insert the middleware after the `app = FastAPI(...)` line and before the route functions.

```bash
@app.middleware("http")
async def security_headers(request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["Cross-Origin-Resource-Policy"] = "same-origin"
    return response
```

The complete file should contain the imports, app definition, middleware, and both endpoint functions. Save the file. If Uvicorn was started with its default settings, stop it with Ctrl+C and start it again; alternatively, run Uvicorn with `--reload` during development.

### Verify headers

```bash
curl -i http://127.0.0.1:8000/
```

Look for `X-Content-Type-Options: nosniff` and `Cross-Origin-Resource-Policy: same-origin`. Earlier, `curl -I` returned HTTP 405 because it sends a HEAD request, while the practice endpoint explicitly supports GET. This was not, by itself, evidence of a security vulnerability. Use `curl -i` for a GET request.

### Optional cache-control headers used in the lab

A later scan reported a cache-related alert on API content. We discussed adding these headers in the same middleware if responses should not be stored by caches:

```bash
response.headers["Cache-Control"] = "no-store"
response.headers["Pragma"] = "no-cache"
response.headers["Expires"] = "0"
```

Place those lines before `return response`. Use `no-store` thoughtfully: it is appropriate for sensitive responses, but public, cacheable content may have different caching requirements. The conversation indicated the cache-related alert changed to Non-Storable Content, but the full latest report was not provided, so verify your own report rather than assuming an exact final alert count.


## 7. Re-run ZAP after changes

Run the same scan again. Use a different report filename so the first report is preserved:

```bash
sudo docker run --rm \\
  --network host \\
  -v "$HOME/zap-reports:/zap/wrk/:rw" \\
  ghcr.io/zaproxy/zaproxy:stable \\
  zap-api-scan.py \\
  -t http://127.0.0.1:8000/openapi.json \\
  -f openapi \\
  -r /zap/wrk/zap-report-final.html
```

The user later shared output with `FAIL-NEW: 0`, `WARN-NEW: 0`, `INFO: 0`, and `PASS: 119`, including PASS checks for loosely scoped cookies, cloud metadata exposure, server-side template injection, and time-based remote OS command injection. This is a summary of that run only; it does not mean every possible test was performed or that the API is fully secure.

A client-error informational message can occur when ZAP probes paths that this small API does not implement and receives 404 responses. A 404 on an undefined path is expected; inspect the exact URL and response before deciding whether it represents an issue.


## 8. Semgrep: scan the source code

Run Semgrep from the project directory so the target file path resolves correctly.

```bash
cd ~/api-lab
source .venv/bin/activate
semgrep --version
semgrep scan --config auto app.py
```

`--config auto` chooses a suitable rule configuration and may need internet access. In the conversation, it ran 290 rules and returned 0 findings on app.py.

Then run an explicit Python ruleset:

```bash
semgrep scan --config p/python app.py
```

In the conversation, this ran 151 Python rules and returned 0 findings on app.py. The output also noted that additional Pro rules were unavailable because the user was not logged in. That message is about product/rule coverage, not a count of vulnerabilities.

The scan output said `only git-tracked` for the initial target in one run. If Semgrep does not scan a file because it is ignored or not tracked, check the target and ignore configuration. Do not assume a zero-finding result is meaningful unless the file was actually scanned.


## 9. Create a deliberately insecure practice example

To verify that Semgrep can flag a known risky pattern, a separate file named semgrep_practice.py was created. This is a local educational example only. Do not execute attacker-controlled input as a command.

```bash
cd ~/api-lab
nano semgrep_practice.py
```

Initial intentionally insecure example:

```bash
import subprocess

def run_command(user_input):
    subprocess.run(user_input, shell=True)
```

Save with Ctrl+O, Enter, Ctrl+X, then scan:

```bash
semgrep scan --config p/python semgrep_practice.py
```

Semgrep detected `subprocess.run` with `shell=True` and marked the finding as Blocking. The rule warned that shell execution can make malicious command execution easier. Treat this as a potential command-injection risk if the input can be controlled by an attacker; the actual exploitability depends on context.


## 10. Fix and verify the practice example

Replace the practice example with a fixed command and a list of separate arguments. This demonstrates avoiding shell interpretation; it is not intended to process user-provided commands.

```bash
import subprocess

def list_directory():
    result = subprocess.run(
        ["ls", "-la"],
        shell=False,
        capture_output=True,
        text=True,
        check=True,
    )
    return result.stdout
```

Save the file. Do not run this example unless you understand that it executes the fixed `ls -la` command. Then scan it again:

```bash
cd ~/api-lab
semgrep scan --config p/python semgrep_practice.py
```

The subsequent scan returned 0 findings across 151 Python rules. This indicates that the enabled rules did not report the previous pattern after the change; it does not prove that the file or application is free of all security issues.


## 11. Save Semgrep JSON evidence

The following command saves machine-readable scan output for later review or audit evidence:

```bash
cd ~/api-lab
mkdir -p security-reports
semgrep scan --config p/python semgrep_practice.py --json --output security-reports/semgrep-report.json
```

Check that the file exists and inspect it:

```bash
ls -lh security-reports/
python3 -m json.tool security-reports/semgrep-report.json | less
```

Press `q` to exit `less`. If the scan command exits with an error, inspect the terminal output and confirm that the path exists. Keep reports free of sensitive code, tokens, and private system details before sharing them.


## 12. Troubleshooting guide

### A. `Invalid scanning root: semgrep_practice.py`

Cause: command was run from `~` instead of `~/api-lab`, so Semgrep could not find the file at that relative path.

```bash
cd ~/api-lab
ls -l semgrep_practice.py
semgrep scan --config p/python semgrep_practice.py
```

### B. `curl -I` returns HTTP 405

`curl -I` sends a HEAD request. The sample endpoint explicitly defines GET routes, so a 405 can be expected. Try a GET request:

```bash
curl -i http://127.0.0.1:8000/
```

### C. Connection refused / API does not respond

Ensure Uvicorn is still running in another terminal, and check the port and URL:

```bash
curl -i http://127.0.0.1:8000/
ss -ltnp | grep 8000
```

If the server is stopped, activate the environment and start it again:

```bash
cd ~/api-lab
source .venv/bin/activate
uvicorn app:app --host 127.0.0.1 --port 8000
```

### D. ZAP cannot reach the API or the scan fails

Check the API is running and the OpenAPI JSON is reachable first:

```bash
curl -sS http://127.0.0.1:8000/openapi.json
```

Then confirm Docker works and the image is available:

```bash
sudo docker run --rm hello-world
docker image ls | grep zaproxy
```

The `--network host` command above assumes a Linux host such as Kali. Docker Desktop on Windows/macOS uses different networking considerations; do not blindly copy the host-network command to those platforms.

### E. Report file not found

```bash
ls -lh ~/zap-reports
ls -lh ~/api-lab/security-reports
```

Use the exact filename shown by `ls`. `xdg-open` only opens an existing file.

### F. Semgrep says no findings

Check that the intended file was scanned, the scan completed successfully, and the rule configuration is appropriate. Zero findings is not a security certification. The `p/python` ruleset in this lab reported 151 rules, and unauthenticated use did not include all Pro rules.

### G. `pip` or virtual-environment issue

```bash
cd ~/api-lab
source .venv/bin/activate
python --version
python -m pip --version
```

If `.venv` does not exist, create it with `python3 -m venv .venv`. Avoid using `sudo pip install` inside the virtual environment.


## 13. Command reference: quick repeat checklist

```bash
# 1. Enter the project and activate its environment
cd ~/api-lab
source .venv/bin/activate

# 2. Start the API (run in Terminal 1)
uvicorn app:app --host 127.0.0.1 --port 8000

# 3. Verify the API (run in Terminal 2)
curl -i http://127.0.0.1:8000/
curl -sS http://127.0.0.1:8000/openapi.json

# 4. Create report folder
mkdir -p ~/zap-reports

# 5. Run ZAP API scan
sudo docker run --rm --network host \\
  -v "$HOME/zap-reports:/zap/wrk/:rw" \\
  ghcr.io/zaproxy/zaproxy:stable \\
  zap-api-scan.py \\
  -t http://127.0.0.1:8000/openapi.json \\
  -f openapi \\
  -r /zap/wrk/zap-report-final.html

# 6. Open ZAP report
xdg-open ~/zap-reports/zap-report-final.html

# 7. Run Semgrep against the API source
semgrep scan --config p/python app.py

# 8. Run Semgrep against the practice example
semgrep scan --config p/python semgrep_practice.py

# 9. Save Semgrep JSON evidence
mkdir -p security-reports
semgrep scan --config p/python semgrep_practice.py --json --output security-reports/semgrep-report.json

# 10. Verify saved files
ls -lh ~/zap-reports
ls -lh ~/api-lab/security-reports
```

The API terminal must remain running during the ZAP scan. The Semgrep commands do not require the API to be running because Semgrep scans source files.


## 14. Results and what to say in a review

Observed results from this lab session:

- ZAP first reported missing X-Content-Type-Options and Cross-Origin-Resource-Policy headers. Middleware was added to set them.
- The later ZAP output shared by the user showed 0 new FAIL and 0 new WARN alerts, 0 INFO entries, and 119 PASS entries. This is only the scope of that run.
- Semgrep `--config auto app.py`: 0 findings; 290 rules ran.
- Semgrep `--config p/python app.py`: 0 findings; 151 Python rules ran.
- Semgrep detected `subprocess.run(..., shell=True)` in the intentionally insecure practice file as a Blocking finding.
- After changing the practice example to a fixed argument list and `shell=False`, the subsequent scan reported 0 findings across 151 Python rules.
Example explanation for a teammate: “I created a local FastAPI practice service, used its OpenAPI specification to run an OWASP ZAP API scan, reviewed the findings, added selected response headers, and reran the scan. I also used Semgrep for static analysis, validated detection with an intentionally risky subprocess example, changed the example to avoid shell execution, and reran Semgrep. Results are limited to the specific endpoints, source files, rules, and tests used.”

For a stronger assessment, add authentication and authorization testing, input validation tests, dependency scanning, manual review, and broader authorized test coverage. Do not represent this small lab as a complete penetration test.

