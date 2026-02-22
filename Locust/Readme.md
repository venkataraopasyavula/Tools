### What is Locust?

Locust is an open-source load testing tool written in Python. It allows you to simulate thousands of concurrent users to test the performance and scalability of your applications, such as web services or APIs. Unlike some traditional tools (e.g., JMeter), Locust uses code-based scripting (in Python) to define user behaviors, making it highly flexible and easy to integrate into CI/CD pipelines. It's particularly useful for testing microservices because you can model complex user flows, like API calls across multiple services.

Key features:
- **Distributed testing**: Run tests across multiple machines for large-scale simulations.
- **Real-time monitoring**: Web-based UI to view stats like requests per second (RPS), response times, and failure rates.
- **Extensible**: Supports HTTP/HTTPS, WebSockets, gRPC, and custom protocols via plugins.
- **Lightweight**: No GUI scripting; everything is code.

For a Java Spring Boot microservices project, Locust treats your app as a black box—it sends requests to your endpoints (e.g., REST APIs) and measures responses. It doesn't require any changes to your Spring Boot code.

### How to Use Locust for a Spring Boot Microservices Project

#### Step 1: Prerequisites
- Python 3.6+ installed.
- Your Spring Boot microservice running locally or on a server (e.g., at `http://localhost:8080`).
- Basic knowledge of Python (Locust scripts are simple).

#### Step 2: Installation
Install Locust via pip:
```
pip install locust
```
For distributed testing or advanced features, you might also install `gevent` or other dependencies, but the base install is sufficient for starters.

#### Step 3: Create a Locust Script (locustfile.py)
This is the core of Locust. You define a "User" class that simulates behavior, such as making HTTP requests to your Spring Boot endpoints.

Create a file named `locustfile.py` in your project directory. Here's a basic structure:
```python
from locust import HttpUser, task, between

class MicroserviceUser(HttpUser):
    # Host is the base URL of your Spring Boot app
    host = "http://localhost:8080"  # Change to your service URL
    
    # Wait time between tasks (simulates user think time)
    wait_time = between(1, 3)  # Seconds
    
    @task(2)  # Weight: This task runs twice as often as others
    def get_user_profile(self):
        # Simulate GET request to a /users/{id} endpoint
        self.client.get("/users/1")
    
    @task(1)  # Weight: Runs half as often
    def create_order(self):
        # Simulate POST request to a /orders endpoint
        payload = {"productId": 123, "quantity": 2}
        self.client.post("/orders", json=payload)
    
    # Add more tasks for other endpoints, e.g., authentication, search, etc.
```
- **`HttpUser`**: Base class for HTTP-based testing.
- **`@task`**: Decorates methods that represent user actions. The number in parentheses is the weight (higher means more frequent execution).
- **`self.client`**: Built-in HTTP client to make requests.
- You can add headers, auth (e.g., JWT), or handle responses (e.g., check status codes).

For microservices, you could have multiple User classes to simulate traffic across services (e.g., one for auth service, one for payment).

#### Step 4: Run Locust
From the directory with `locustfile.py`, run:
```
locust
```
- This starts a web server at `http://localhost:8089`.
- Open it in your browser, enter the number of users to simulate (e.g., 100), hatch rate (e.g., 10 users/second), and host (if not set in the script).
- Click "Start swarming" to begin the test.
- Monitor real-time charts for RPS, average/median response times, percentiles (e.g., 95th), and failures.

To run headlessly (e.g., in CI/CD):
```
locust --headless --users 500 --spawn-rate 20 --run-time 5m
```
- `--users`: Total simulated users.
- `--spawn-rate`: How fast to ramp up users.
- `--run-time`: Duration (e.g., 5m for 5 minutes).

#### Step 5: Analyze Results
- Locust's UI shows live stats.
- Export CSV reports via the UI or command line (`--csv=results`).
- Look for bottlenecks: High response times might indicate database issues in your Spring Boot app; failures could be rate limiting or errors.

#### Step 6: Advanced Tips for Spring Boot Microservices
- **Authentication**: If your APIs use OAuth/JWT, add it in the User class:
  ```python
  def on_start(self):
      # Login once per user
      response = self.client.post("/login", json={"username": "test", "password": "pass"})
      self.token = response.json()["token"]
  
  @task
  def protected_endpoint(self):
      self.client.get("/protected", headers={"Authorization": f"Bearer {self.token}"})
  ```
- **gRPC Support**: For Spring Boot gRPC services, install `locust-plugins` (`pip install locust-plugins`) and use `GrpcUser`.
- **Distributed Mode**: For large tests, run a master (`locust -f locustfile.py --master`) and workers (`locust -f locustfile.py --worker --master-host=localhost`).
- **Integration with Spring Boot**: No direct integration needed, but you can monitor Spring Boot metrics (e.g., via Actuator) alongside Locust.
- **Best Practices**:
  - Start small: Test with 10-50 users to baseline.
  - Model realistic scenarios: Mix GET/POST, vary payloads.
  - Handle failures: Use `catch_response=True` to mark successes/failures based on custom logic.
  - Scale: Use Docker/Kubernetes for distributed Locust if testing cloud-deployed microservices.

### Example: Testing a Simple Spring Boot Microservice

Assume you have a Spring Boot app with two endpoints:
- `GET /api/greeting`: Returns "Hello, World!"
- `POST /api/echo`: Echoes back the JSON payload.

**locustfile.py**:
```python
from locust import HttpUser, task, between

class GreetingUser(HttpUser):
    host = "http://localhost:8080/api"  # Base path to your microservice
    wait_time = between(0.5, 2.0)
    
    @task(3)
    def get_greeting(self):
        with self.client.get("/greeting", catch_response=True) as response:
            if response.status_code != 200:
                response.failure("Greeting failed")
            elif "Hello" not in response.text:
                response.failure("Unexpected response")
    
    @task(1)
    def post_echo(self):
        payload = {"message": "Test load"}
        self.client.post("/echo", json=payload)
```

Run `locust`, simulate 200 users at 5/sec, and observe how your Spring Boot app handles the load. If response times spike, check for optimizations like caching or database tuning.

we can check the official Locust docs at locust.io for more info.
