### Token Bucket Algorithm in a Distributed AWS Lambda Environment

The **token bucket algorithm** is crucial for network traffic management, as it controls the rate of packet transmission over a network based on token availability. Each token represents the right to transmit a certain volume of data. Tokens are added to a bucket at a consistent rate, never surpassing the bucket's limit. If the bucket is full, incoming packets must either wait for new tokens to be available or be dropped.

Now, applying this concept to a distributed environment with AWS Lambda, we can design a solution to integrate with our vendors while keeping the throughput below the stipulated TPS (Transactions Per Second), offering a low-cost and scalable solution.

---

### Distributed Token Bucket Model

In this model, an AWS Lambda function named **"vendor-outbound-gateway-lambda-service"** is responsible for making requests to the vendor's REST APIs.

#### Infrastructure Components:
- **SQS Queue**: `vendor-outbound-gateway-service-request-channel`
- **SQS Dead Letter Queue (DLQ)**: `vendor-outbound-gateway-service-request-channel-dlq`
- **SNS Topic**: `vendor-outbound-gateway-service-reply-channel`

#### Application Components:
- **AWS Lambda**: `vendor-outbound-gateway-service`

All communication between the system and the vendor occurs through this Lambda service. Messages are passed through the SQS queue, and responses are handled via the SNS topic. Large payloads are stored in S3, and only the object keys are passed through the channels.

The vendor imposes a rate limit of **10 TPS**. To stay within this limit, the token bucket algorithm can be employed where **each Lambda instance represents a token**. Each instance has a fixed runtime (AWS Lambda timeout), and tokens (Lambda instances) are never generated beyond the system's capacity (concurrency). As instances finish processing, new tokens (Lambda instances) are made available.

---

### Configuring the Token Bucket

Each token can process up to 10 requests, which corresponds to an SQS batch size of 10. For example, with an AWS Lambda concurrency setting of 1 (i.e., 1 token), each token can handle 10 requests at a time.

Increasing the concurrency to 2 tokens allows for 20 requests to be processed, and so on. However, this model assumes that each Lambda instance completes execution in exactly 1 second, enabling the processing of 10 requests per second (per instance). 

But real-world factors like latency complicate this. For example, if each request takes **5 seconds** plus additional overhead for Lambda cold starts, deserialization, and response processing, the total processing time could reach **10 seconds**. In this case, **10 requests every 10 seconds** means that the throughput is effectively **1 TPS** per token. To achieve **10 TPS**, you would need **10 tokens**.

---

### Calculating Visibility Timeout and Token Availability

In this scenario, the ideal visibility timeout for the SQS queue would be at least **15 seconds**. This ensures that if a batch of messages is polled but no token is available, the messages will return to the queue after 15 seconds, allowing time for a new token to be released. Messages that are in flight will be processed within this window, ensuring that no duplicate messages are sent before their current processing is complete.

### Throughput Measurement

- Each token performs a fixed number of requests to the vendor (up to 10 per token).
- Each token has **X seconds** to process and respond before the visibility timeout forces a reattempt.
  
Throughput is calculated as:

\[
\text{Throughput} = \frac{\text{requests per token} \times \text{number of tokens available}}{\text{time until next token is available}}
\]

For example, if each token processes 10 requests in 5 seconds, the throughput would be:

\[
10 \times 5 \text{ (tokens)} = 50 \text{ requests} / 10s = 5 TPS
\]

---

### Problems with this Solution

1. **Variable Latency and Unpredictability**:
   Factors like Lambda cold starts, network delays, or vendor response times can cause requests to take longer or shorter than expected, leading to fluctuating throughput.

   **Potential Solutions**:
   - **Backoff retries**: Implement exponential backoff to avoid overloading the vendor with retries.
   - **Integration timeouts**: Based on the vendor's average response time (e.g., 2 seconds), enforce request timeouts using:
     ```python
     requests.get("vendor API address", timeout=2)
     ```
   - **Dynamic rate limiting**: Adjust the rate of requests dynamically, based on real-time vendor response times.

2. **Exceeding Vendor Rate Limits Due to Faster Token Release**:
   If tokens (Lambda instances) finish faster than anticipated, you risk breaching the vendor's **10 TPS** limit.

   **Potential Solutions**:
   - Calculate both the maximum and minimum possible throughput and configure the system to always stay below the vendor’s TPS limit. 

   For example, if the maximum token release is 1 second and the minimum is 10 seconds, you get:
   - **Max throughput**: 10 tokens \* 10 requests / 1s = **100 TPS**
   - **Min throughput**: 10 tokens \* 10 requests / 10s = **10 TPS**

   Adjust the system to cap at a safe level (e.g., **2 tokens**, **2 requests per token**) to ensure you stay below the limit.

3. **Long Queue Time and Message Duplication**:
   Delays may occur if no tokens are available, leading to processing delays or even duplicated messages.

   **Potential Solutions**:
   - **Adaptive concurrency**: Dynamically adjust the number of tokens based on system load and vendor limits.
   - **Idempotency**: Implement idempotency in the queue or database to prevent duplicate processing.

4. **Handling Vendor Feedback**:
   The vendor may return 429 (Too Many Requests) or retry-later headers, which need to be handled gracefully.

   **Potential Solutions**:
   - Implement **backoff retries** or pause processing using **circuit breakers** to avoid overwhelming the vendor.

---

By refining this approach with dynamic rate limiting, backoff strategies, and appropriate timeout configurations, you can ensure smooth and efficient integration with the vendor while staying within their rate limits.