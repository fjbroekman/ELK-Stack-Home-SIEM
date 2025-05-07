<h1 align="center">🛡️ ELK Stack Home SIEM Lab</h1>

Welcome to my personal Security Information and Event Management (SIEM) lab built using the ELK Stack (Elasticsearch, Logstash, Kibana) along with Filebeat for log shipping. This project simulates a centralized logging environment for monitoring and analyzing system logs, making it a great practice setup for cybersecurity learning and blue team operations.


---

<h2 align="center">📸 Project Overview</h2>

This lab is designed for:

- Practicing log analysis and incident detection
- Understanding how logs flow from endpoints to a SIEM
- Building custom dashboards and visualizations
- Learning how to detect suspicious behavior using Kibana

---

<h2 align="center">🧰 Stack Components</h2>

| Component     | Description                               |
|---------------|-------------------------------------------|
| Elasticsearch | Stores and indexes log data               |
| Kibana        | Visualizes data and builds dashboards     |
| Filebeat      | Collects and ships logs from endpoints    |

---

<h2 align="center">🏗️ Architecture</h2>


1. Elasticsearch, Kibana and Filebeat is installed on a Linux host
2. Filebeat ships logs to Elasticsearch.
3. Kibana is used to analyze and visualize the collected logs.
4. Kibana is secured using Certificate Authorities.
   
---

## 🚀 Getting Started


