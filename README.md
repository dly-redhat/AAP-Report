# Red Hat Lightspeed host and vulnerability report

One playbook and one Jinja2 template. The report lists every server in the
Red Hat Lightspeed inventory with its OS, last check-in, and its critical,
important and known-exploit CVE counts. It has a **Download PDF** button.

Nothing connects to the servers. The job runs on the AAP execution node and only
talks to `sso.redhat.com` and `console.redhat.com`, so no SSH credential is needed.

## Files

| File | Purpose |
| --- | --- |
| `lightspeed_report.yml` | Gets a token, asks Lightspeed for each host's data, renders the report |
| `templates/lightspeed_report.html.j2` | The HTML page: totals at the top, one row per server |

Copy both into the Git repository your AAP project uses, keeping the
`templates/` folder next to the playbook.

## How it works

1. **Hosts** come from the inventory you already built with the Red Hat
   Lightspeed (Insights) inventory source. That source gives each host an
   `insights_id` variable: its Lightspeed ID.
2. **A token** is requested from `sso.redhat.com` with the service account
   (`INSIGHTS_CLIENT_ID` / `INSIGHTS_CLIENT_SECRET`).
3. **For each host**, the vulnerability API is asked for:
   - the system record: OS and last upload time;
   - four counts: critical CVEs, important CVEs, CVEs with a known exploit, and
     CVEs with a fix available.
4. **The template** turns those rows into the HTML report.

A host with no vulnerability data in Lightspeed is still listed, marked
"No vulnerability data", and is never counted as safe.

## Set up in AAP

### 1. Credential

The job needs the service account in two environment variables:
`INSIGHTS_CLIENT_ID` and `INSIGHTS_CLIENT_SECRET`.

- If the credential you made for the inventory source already injects those two
  variables, reuse it.
- Otherwise create a credential type (**Infrastructure > Credential Types >
  Create**), named for example `Red Hat Lightspeed service account`:

  Input configuration:

  ```yaml
  fields:
    - id: client_id
      type: string
      label: Client ID
    - id: client_secret
      type: string
      label: Client secret
      secret: true
  required:
    - client_id
    - client_secret
  ```

  Injector configuration:

  ```yaml
  env:
    INSIGHTS_CLIENT_ID: '{{ client_id }}'
    INSIGHTS_CLIENT_SECRET: '{{ client_secret }}'
  ```

  Then create a credential of that type with the service account's client ID
  and secret.

### 2. Job template

| Field | Value |
| --- | --- |
| Inventory | The Lightspeed inventory |
| Project | The project that holds these files |
| Playbook | `lightspeed_report.yml` |
| Credentials | The Lightspeed service account credential (no machine credential needed) |
| Extra variables | See below |

### 3. Where the report is saved

By default the file is written to `/tmp/lightspeed_report.html` inside the job's
container, which disappears when the job ends. For real use, save it on a Linux
server that can serve or share it:

```yaml
lightspeed_report_host: reports01.example.com   # a host in the job's inventory
lightspeed_report_dest: /var/www/html/lightspeed_report.html
```

That host needs to be in the job's inventory, and the job needs a machine
credential for it.

## Variables you may change

| Variable | Default | Purpose |
| --- | --- | --- |
| `lightspeed_report_title` | `Server vulnerability report` | Report heading |
| `lightspeed_report_host` | `localhost` | Host that stores the HTML file |
| `lightspeed_report_dest` | `/tmp/lightspeed_report.html` | Path of the HTML file |
| `lightspeed_report_hosts` | every host in the inventory | Hosts to report on |
| `lightspeed_report_timeout` | `90` | Seconds to wait for each Lightspeed call |

## Notes

- The service account needs read access to the Vulnerability service in
  console.redhat.com (*User Access*: the **Vulnerability viewer** role).
- The job makes five API calls per host. Lightspeed can be slow, so each call
  waits up to 90 seconds and retries three times.
- If the inventory source uses a different `vars_prefix` than the default
  `insights_`, change `insights_id` in the playbook to match.
