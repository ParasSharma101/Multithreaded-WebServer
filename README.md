# Multithreaded Web Server Project

This project demonstrates three types of Java web servers:
- Single-threaded
- Multithreaded
- Thread pool-based

## Project Structure

```
Multithreaded/
    Client.java
    Server.java
SingleThreaded/
    Client.java
    Server.java
ThreadPool/
    Server.java
```

## How to Run

### Prerequisites

- Java JDK (8 or above)
- Apache JMeter (for load testing)

### Compile the Servers

Open a terminal in the project directory and run:

```powershell
# For SingleThreaded server
cd SingleThreaded
javac Server.java Client.java

# For Multithreaded server
cd ..\Multithreaded
javac Server.java Client.java

# For ThreadPool server
cd ..\ThreadPool
javac Server.java
```

### Start a Server

Example for starting the Multithreaded server:

```powershell
cd Multithreaded
java Server
```

The server will start and listen on the default port (usually 8080 or as specified in the code).

### Run a Client (Optional)

You can run the provided `Client.java` to send requests:

```powershell
java Client
```

## Testing with JMeter

1. Download and install [Apache JMeter](https://jmeter.apache.org/).
2. Open JMeter and create a new test plan.
3. Add a Thread Group and set the number of users (threads) and loop count.
4. Add an HTTP Request sampler:
    - Set the server name or IP (e.g., `localhost`)
    - Set the port (e.g., `8080`)
    - Set the path (e.g., `/`)
5. Add a Listener (e.g., View Results Tree) to see the results.
6. Start the server you want to test.
7. Click the Start button in JMeter to begin the test.

## Notes

- Make sure the server is running before starting the JMeter test.
- You can compare the performance of different server implementations by running JMeter tests against each one.
