# SPL Cheat Sheet for SOC Analysts & Threat Hunters

A quick-reference guide for Splunk Search Processing Language (SPL), bridging the gap between Linux command-line text processing (`grep`, `awk`, `sed`) and enterprise SIEM threat detection.

---

## 🧠 Core Philosophy: The Pipeline
Splunk uses a pipeline architecture, identical in concept to Linux pipes (`|`). You take a massive dataset, filter out the noise, extract the exact strings you need, perform mathematical operations, and format the output.

**Golden Rule of SPL:** *Filter Early, Filter Often.* Put your strictest filters (index, sourcetype, specific keywords) before the first pipe to save compute power and reduce search times.

`Search | Filter | Extract | Aggregate | Format`

---

## 🔍 1. Data Filtering (The `grep` Equivalents)
Used to narrow down the dataset before applying heavy mathematical operations.

| Command | Description | Example |
| :--- | :--- | :--- |
| `search` | Implicitly used at the start, or explicitly after a pipe to filter results. | `... \| search status="Failed"` |
| `where` | Evaluates a boolean expression; allows comparing two fields. | `... \| where failed_logins > success_logins` |
| `dedup` | Removes duplicate events based on a specific field. Great for noisy alerts. | `... \| dedup src_ip` |
| `fields` | Keeps (`+`) or removes (`-`) fields to improve search performance. | `... \| fields + _time, src_ip, user` |
| `head` / `tail` | Returns the first (newest) or last (oldest) *N* results. | `... \| head 10` |

---

## 🛠️ 2. Data Manipulation (The `awk` / `sed` Equivalents)
Used to create new data points, fix formatting, or carve out specific text from raw logs.

| Command | Description | Example |
| :--- | :--- | :--- |
| `rex` | Extracts fields on the fly using Regular Expressions. | `... \| rex "user (?<username>[^\s]+)"` |
| `eval` | Calculates or creates new fields using math, strings, or logic. | `... \| eval risk = if(action=="block", 10, 50)` |
| `coalesce` | (Used with `eval`) Takes the first non-null value from a list of fields. | `... \| eval ip = coalesce(src_ip, dest_ip)` |
| `split` | (Used with `eval`) Breaks a string into a multi-value field. | `... \| eval parts = split(domain, ".")` |

---

## 📊 3. Statistics & Aggregation (The SOC Engine)
The core commands used for writing Detection Rules and Correlation Searches.

| Command | Description | Example |
| :--- | :--- | :--- |
| `stats count` | Counts the total number of events. | `... \| stats count by src_ip` |
| `stats dc()` | **Distinct Count:** Counts unique values (e.g., how many *different* users). | `... \| stats dc(user) by src_ip` |
| `stats values()` | Lists all unique values associated with the group. | `... \| stats values(dest_port) by src_ip` |
| `bin` / `bucket` | Groups events into specific time chunks (vital for rate-based attacks). | `... \| bin _time span=5m` |
| `timechart` | Aggregates data specifically for time-series visualizations. | `... \| timechart span=1h count by action` |

> **Example: Basic Brute Force Detection Logic**
> ```splunk
> index=linux_auth sourcetype=syslog "Failed password"
> | bin _time span=1m
> | stats count as failures, dc(user) as unique_users by src_ip, _time
> | where failures > 5
> ```

---

## 🎨 4. Formatting (Dashboards & Reporting)
Used at the very end of the pipeline to make the data readable for L2 analysts, managers, or incident response tickets.

| Command | Description | Example |
| :--- | :--- | :--- |
| `table` | Creates a clean, ordered data table with specified columns. | `... \| table _time, src_ip, user, port` |
| `sort` | Sorts results. Use `-` for descending (highest to lowest). | `... \| sort - failures` |
| `rename` | Changes a field name for better readability in reports. | `... \| rename src_ip as "Attacker IP"` |
| `fillnull` | Replaces empty (null) fields with a specific string (e.g., "N/A"). | `... \| fillnull value="Unknown" user` |

---

## 💡 Pro-Tips for SOC L1 / L2
* **Time is everything:** Always include `_time` in your tables when analyzing an attack sequence.
* **Use `TERM()` for speed:** If looking for an exact IP or string, `TERM(192.168.1.16)` bypasses Splunk's minor word-breaking rules and searches faster.
* **Comments:** Use ``` `comment("Your text here")` ``` to annotate complex SPL queries for your team.
