# Purple Team Capstone: Blue Team

## Put Briefly, By Yours Truly

This "blue team" process is a product of me breaking into (hacking) a network that I designed; that hacking part is called red teaming (link to red team). Now I switch hats.

My goal was to behave as a Level 1 SOC analyst: notice the Slack alert (SOAR), head over to Wazuh (SIEM/EDR) and investigate the particular logs those alerts were GENERATED from, filter by agent name and time, and observe the filtered logs to correlate events and build the final story.

But before we get to that, let me explain what all these tools actually ARE and why they exist.

---

### The Tools

Every machine has the ability to record actions taken on it: places visited, files accessed or downloaded or deleted, settings changed, apps updated. ALL of it. These records are called logs. They're RECEIPTS. And if you want specific receipts for specific actions taken on a computer, say, a hacker accessing your machine and doing things you wouldn't want them to do, you'd check THOSE specific logs.

If you want all this info in a centralized, NICELY formatted place like a dashboard, you need a way of getting those receipts from ONE device to a centralized place. That happens via AGENTS. They're little pieces of software installed on the devices whose info you want forwarded, and they forward what you TELL them to forward. They ALL send their info to a centralized HUB called a MANAGER, who collects everything, normalizes it, and sends it to a DASHBOARD so you can actually read it. Those are the three components of Wazuh: forwarding agent, manager, and dashboard. This dashboard is referred to as a SIEM: Security Information and Event Management system. It's where you read what's going on, investigate, and draw conclusions.

Where does Shuffler come in? Shuffler is a SOAR platform: Security Orchestration, Automation, and Response. If you have a SIEM aggregating actions taken on all these different machines, you may ALSO want a platform to automate the process of informing and responding. These SOAR platforms can email, Slack message, text, notify in maaaany different ways, so the analyst knows to hop on the dashboard and see what's going on. You're orchestrating a system that identifies security anomalies, and AUTOMATING its RESPONSE to those anomalies. That automated response is called a PLAYBOOK: the automatic steps taken when certain things happen. It could isolate a machine from the network, notify an analyst, and automatically start a ticket in the company ticketing software.

Now, how does Wazuh know WHAT to look for? That's detection engineering. You write rules in a big text file that basically tell the computer: "if you see an action performed with these words in the command, or this location, or this pattern, flag it and tell me about it." These rules can be as specific or broad as you like. One of the most common detections is when someone reads, edits, or creates files in a folder called /tmp. The everyday user doesn't interact with /tmp much, but hackers LOVE it because ANYONE can read or write to it without special permissions. They put bad stuff there. So you write a rule, the computer always checks its rules and compares behaviors to them, and if an action matches, it gets flagged in the SIEM.

But rules aren't always perfect. There are exceptions. Some operating system processes touch /tmp legitimately and your rule will flag them anyway. These are called false positives. You don't want your SIEM sending you a million notifications for those exceptions, so you write suppression rules: smaller rules that watch out for the few cases where the behavior is NOT malicious. Then you're only getting alerted on the stuff that actually matters.

Now, how do you know WHAT behaviors to write rules for in the first place? There's a gold standard catalog of attacker tactics and techniques built from real world incidents called MITRE ATT&CK. Security professionals write their baseline detection rules based on the known attacker behaviors listed there, and when a rule fires, they map it to the specific MITRE technique it relates to. That way anybody reading the alert in the SIEM doesn't just know WHAT happened. They know which category of attack behavior it belongs to and what the attacker was likely trying to accomplish. Most modern SIEMs like Wazuh include this MITRE context automatically when you read a log.

---

### So What Did I Do, Exactly?

Once everything was set up, I used a tool called Atomic Red Team to simulate real attacks against my lab machines without manually re-running the entire red team chain every session. Atomic Red Team is a library of small, individual attack simulations, each mapped to a specific MITRE ATT&CK technique. Fire one off, then go investigate it in Wazuh. Exactly like a real analyst on shift. It's really helpful in a case like this where you don't want to MANUALLY conduct every attack that you'd like to investigate in the SIEM.

My Slack channel lit up. Several alerts, all from the same machine, all within a few seconds of each other; it took a lot of alert tuning to reduce false positives on things like /tmp activity. I went to Wazuh and filtered all activity on that machine down to the last 15 minutes. That time window is how you cut through background noise and focus on what happened during the incident. Depending on the info you're given, an analyst would typically filter by device, time, source and destination IP, and some others I don't fully understand yet.

What I saw was a cluster of events firing within the same second. One suspicious event can be noise. Three suspicious events on the same machine in the same second is a STORY.

Buried in the middle of that cluster was one alert that stood out: "Crontab entry changed." A cron job is a scheduled task: you're telling the computer to run a specific command automatically on a repeating schedule. In a legitimate context they're used for backups, updates, maintenance. But in the hands of an attacker who's already gotten in? A cron job is how you establish PERSISTENCE. Write a cron job that re-executes your malicious code every hour, and even if someone restarts the machine, your attack comes right back with it.

That alert told me: someone got in, and they're trying to STAY.

I also saw privilege escalation events in the same timeframe. Sudo commands being run, session opens and closes, a bash shell spawning. Together these events told the story an analyst should stitch together: attacker gained access, escalated privileges to root, and used that root access to write somnething that allows them to stay.

Connecting related events like these to build a timeline and understand what happened is called CORRELATION. From what I've learned, it's the core skill of a SOC analyst, and the ability to know WHAT you're looking at and what it means, and to connect the dots between events, comes with reps like these.

---

### What Comes Next: The IR Lifecycle

Once you've identified and correlated the events, every incident follows the same structure: the Incident Response Lifecycle.

CONTAIN. Stop the bleeding first. Isolate the affected machine from the rest of the network. Still running so you can investigate, but nothing can spread.

ERADICATE. Figure out everything the attacker did and undo it. Remove the malicious cron job. Delete any files they dropped. Close whatever door they used to get in.

RECOVER. Restore the machine to a clean state. This might mean restoring from a known good backup, or in severe cases rebuilding entirely. The recovery method depends on the situation.

DOCUMENT. Write a full incident report: what happened, when, how the attacker got in, what they did, and most importantly what needs to change so it doesn't happen the same way again. This is called root cause analysis, and the recommendations that come out of it are what actually make an organization more secure over time.

---

### What I Built

| Component | Tool | Purpose |
|---|---|---|
| SIEM | Wazuh | Centralized log collection, alert management, investigation |
| SOAR | Shuffler.io | Automated Slack alerting, incident notification |
| Attack simulation | Atomic Red Team | Simulating MITRE ATT&CK techniques to test detection coverage |
| Detection rules | Custom XML | 20+ rules mapped to MITRE ATT&CK |
| Notification channel | Slack | Real-time analyst alerting |

**Detection coverage:**

- Web shell upload and execution (T1505.003)
- Reverse shell via bash (T1059.004)
- Command execution by web server user, RCE indicator (T1505.003)
- Privilege escalation via PwnKit / pkexec (T1548)
- SSH lateral movement (T1021.004)
- Cron job persistence (T1053.003)
- New backdoor user account creation (T1136.001)
- File staging in /tmp (T1036)
- Data exfiltration via netcat and curl (T1041)
- Network scanning tools executed on host (T1046)

---

Next: [Detection Engineering](detection-engineering.md)
