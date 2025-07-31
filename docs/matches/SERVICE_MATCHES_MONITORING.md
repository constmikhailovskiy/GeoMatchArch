### 📊 Business Metrics
Business metrics represent how successfully the Matching service performs its business function — helping people find potential partners or friends based on their interests and visited locations.

#### Match Requests
- **Description:** The total number of match search requests over a queried period. This is the primary indicator of user activity.
- **Alerting:** "Notify if the number of requests in the last hour drops by >50% compared to the same time last week." This could indicate problems with the client application or that users are losing interest in the matching feature.

#### Match-to-Action Rate
- **Description:** The percentage of profile views that result in an action (e.g., starting a private message conversation). This is a key metric for the quality of the matching algorithm.
- **Alerting:** This is more of an analytical metric, but an alert can be configured if this rate drops sharply, which could indicate a degradation in the matching algorithm.

---

### 🚀 Application/Performance Metrics

#### Response Time / Latency
This is the most important technical metric for this service. It measures the time from receiving a request to sending a response. We want to measure the response time for the main and most critical request of this service: **`GET /api/v1/matches`**. This is the request the client application sends when a user wants to see new potential partners. The speed of this request directly impacts how quickly the user sees new profiles to "swipe." It's the most frequent, critical, and load-intensive operation in the entire service, making its performance crucial for the user experience.

We track this metric using percentiles:
- **p50 (Median):** The response time for the average user.
- **p95:** The upper bound of response time for 95% of users.
- **p99:** The response time for the slowest requests, observed by 1% of users. Helps in identifying outlier problems.

- **Alerting:** "Critical alert if **p99 response time > 500ms** for 5 minutes." This means a significant portion of users are having a very slow experience, which could lead to churn.

#### Error Rate
- **Description:** The percentage of requests that result in a server error (5xx codes).
- **Alerting:** "Critical alert if **error rate > 1%** for 1 minute." Even a small error rate can affect thousands of users under high load.

#### Database Response Time
- **Description:** How much time Elasticsearch spends executing a search query. This helps isolate problems: if the service response time increases, we can see if the database or the application itself is the culprit.
- **Alerting:** "Notify if **p95 Elasticsearch query latency > 150ms**." This could indicate inefficient queries or problems with the Elasticsearch cluster.

---

### ⚙️ System/Infrastructure Metrics
These metrics show the health of the infrastructure on which the service runs.

#### CPU Utilization
- **Description:** The percentage of processor usage on the servers where the service is running.
- **Alerting:** "Notify if **CPU utilization > 80%** for 10 minutes." This is a signal that scaling is needed (e.g., adding new nodes).