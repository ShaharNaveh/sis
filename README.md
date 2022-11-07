# Tasks

## Task 2

Start the `Sample ECommerce` elasticube:

<details open>
<summary>command</summary>

```bash
si elasticubes start -name Sample\ ECommerce
```

</details>

Build the `Sample ECommerce` elasticube:

<details open>
<summary>command</summary>

```bash
si elasticubes build -name Sample\ ECommerce -type full
```

</details>

---

## Task 3

**Results file located at**: `/home/ubuntu/result.csv`

<details>
<summary>task_3.py</summary>

```python
#!/usr/bin/env python3
import csv
import datetime
import os
import re
import shlex
import subprocess

import requests  # Requests already installed on the system's python evironment

NS = "sisense"
KUBECTL = "/usr/local/bin/kubectl"
K = f"{KUBECTL} -n {NS}"

ADMIN_USER = os.getenv("CRON_ADMIN_USER", "44@sisense.com")
SISENSE_HOST = os.getenv("CRON_SISENSE_HOST", "https://shaharsisense.sisense.com")
ADMIN_PASSWORD = os.environ["CRON_ADMIN_PASSWORD"]
OUTFILE = os.getenv("CRON_OUTPUT", "/home/ubuntu/result.csv")

NOW = datetime.datetime.utcnow()


def get_rest_email(host: str, login_username: str, login_password: str) -> str:
    data = {"username": login_username, "password": login_password}
    with requests.Session() as session:
        resp = session.post(f"{host}/api/v1/authentication/login", data=data)
        if resp.status_code not in [200, 201, 204]:
            raise Exception(f"ERROR: {resp.status_code}: {resp.content}") from None

        access_token = resp.json()["access_token"]
        headers = {"authorization": f"Bearer {access_token}"}
        resp = session.get(f"{host}/api/users/", headers=headers)
    users = resp.json()
    for user in users:
        if user.get("email", "") != ADMIN_USER:
            continue
        return user["userName"]


def get_mongo_pod() -> str:
    cmd = f"{K} get pods -l app.kubernetes.io/component=mongodb -ojsonpath='{{.items[0].metadata.name}}'"
    output = subprocess.check_output(shlex.split(cmd))
    return output.decode()


def get_mongo_email(*, pod: str, username: str) -> str:
    query = re.escape(
        "db.getSiblingDB('prismWebDB').users.findOne({'userName': "
        + repr(username)
        + "}, {'email': true, '_id': false})"
    )
    cmd = f'''{K} exec {pod} -c mongodb -- bash -c "echo {query} | mongo --quiet"'''

    output = subprocess.check_output(cmd, shell=True).decode()
    email = re.findall(r"[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+", output)[0]

    return email


def main() -> None:
    pod = get_mongo_pod()
    mongo_email = get_mongo_email(pod=pod, username=ADMIN_USER)
    rest_email = get_rest_email(
        host=SISENSE_HOST, login_username=ADMIN_USER, login_password=ADMIN_PASSWORD
    )
    status = mongo_email == rest_email
    result = {
        "Time": str(NOW),
        "MongoDB": mongo_email,
        "Rest": rest_email,
        "Status": status,
    }

    with open(OUTFILE, mode="a+", newline="") as csvfile:
        writer = csv.DictWriter(csvfile, fieldnames=result.keys())
        writer.writerow(result)

    exit(not status)


if __name__ == "__main__":
    main()
```

</details>

### Task3.b

<details open>
<summary>task3.timer</summary>

```desktop
# /home/ubuntu/.config/systemd/user/task3.timer
[Unit]
Description=Task 3 timer

[Timer]
OnUnitActiveSec=1h

[Install]
WantedBy=timers.target
```

</details>

<details open>
<summary>task3.service</summary>

```desktop
# /home/ubuntu/.config/systemd/user/task3.service

[Unit]
Description=Task 3

[Service]
Type=oneshot
ExecStart=/usr/local/bin/task_3.py
ExecStopPost=/bin/bash -c 'if [ "$$EXIT_STATUS" -ne 0 ]; then INTERVAL="30m"; else INTERVAL="1h" ; fi ; /bin/sed -i -E "/OnUnitActiveSec=/{s/=.+/=$INTERVAL/}" /home/ubuntu/.config/systemd/user/task3.timer && systemctl --user daemon-reload && systemctl --user restart task3.timer'

[Install]
WantedBy=default.target
```

</details>


<details open>
<summary>override.conf</summary>

```desktop
# /home/ubuntu/.config/systemd/user/task3.service.d/override.conf

[Service]
Environment="CRON_ADMIN_PASSWORD=PASSWORD"
```

</details>
