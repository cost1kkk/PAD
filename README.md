# Guess the Country by Flag App Platform

This app is a game where users guess the name of countries based on their flags. Users can register, participate in the quiz, and view their rankings on a live leaderboard. The platform leverages microservices for separate functionalities like user registration, the flag guessing game with a live leaderboard, ensuring scalability and real-time updates using Websockets.

# Assess Application Suitability
1. Easy Communication
- Real-time interaction enables multiple users to guess flags simultaneously, with the leaderboard updating live as they play.
2. Easy Availability
- The architecture ensures that if for example ”User Registration Service” experiences issues, the ”Game Service” will continue to function normally and other users should be able to play.
3. Scalability
- By utilizing microservices, components such as Game Service and User Registration Service can scale independently, allowing the app to handle large numbers of users.


Real-world Examples similar to my app:
- Kahoot! is a popular online quiz platform where users can create and participate in quizzes on various topics, with real-time interaction, scoring, and leaderboards.
- HQ Trivia is a live quiz app where users could join and answer trivia questions in real-time, competing for cash prizes with live leaderboard updates


# Service Boundaries

User Registration Service - Manages the registration and login processes for users, allowing them to create accounts and log in to participate in the game.
Game Service - Handles the core gameplay where users are shown a flag and must guess the corresponding country, with points awarded for correct answers.

# Technology Stack and Communication Patterns

1. User Registration Service
    - Language: Python
    - Framework: Flask
    - Communication: REST API for interaction with the Game and Leaderboard services
2. Game Service
    - Language: Python
    - Framework: Flask-SocketIO
    - Communication: Websockets for real-time interactions during gameplay
3. Gateway & Service Discovery
    - Language: Java
    - Framework: Spring Cloud Gateway and Eureka for service discovery
    - Communication: Routes HTTP requests to the correct microservices

Diagram for classification:

![The diagram](/pad1.png)

# Design Data Management

## User Registration Service

1. Register a new user (POST /user/register)

- Input

```
    {
        "username": "cost1k",
        "password": "imiiubescmama",
        "email": "costik@mail.ru"
    }
```

- Output

```
    {
        "message": "Account successfully created! You can start playing."
    }
```

2. Login (POST /user/login)

- Input

```
    {
        "username": "cost1k",
        "password": "imiiubescmama"
    }
```

- Output

```
    {
        "message": "I was sure you will come back stronger!"
    }
```

## Game Service

1. Start a new game (POST /game/start)

- Input

```
    {
        "userId": "user_12345"
    }

```

- Output

```
    {
        "message": "Game started! Identify the country of this flag.",
        "flag_image_url": "https://flag-image-url.com/flag.png"
    }
```
2. Submit a guess (POST /game/guess)

- Input

```
    {
        "userId": "user_12345"
        "guess": "Brazil"
    }

```

- Output

```
    {
        "correct": true,
        "points_earned": 10,
        "message": "Correct! You’ve earned 10 points."
    }
```
3. View live leaderboard (GET /leaderboard):
- Output

```
  {
        "leaderboard": [
            {"username": "player1", "points": 150},
            {"username": "player2", "points": 130}
        ]
    }
```

# Set up Deployment and Scaling

Each microservice will have its own Dockerfile to allow for independent deployment and scaling.

Game Service Dockerfile:
```
FROM python:3.9-alpine
WORKDIR /app
COPY . .
RUN pip install flask flask-socketio
CMD ["python", "app.py"]


```

User Registration Service Dockerfile

```
FROM python:3.9-alpine
WORKDIR /app
COPY . .
RUN pip install flask
CMD ["python", "app.py"]

```

Example of Docker Compose

```
     version: '3.8'
services: 
  user-registration-service:
    build: ./user-registration-service
    ports:
      - "5000:5000"

  game-service:
    build: ./game-service
    ports:
      - "5001:5001"

  redis:
    image: "redis:alpine"

  gateway:
    build: ./gateway
    ports:
      - "8080:8080"

  eureka:
    image: "eureka-server:latest"
    ports:
      - "8761:8761"


```

# Implementation

Both microservices will include a 10-second task timeout for improved responsiveness. The system will handle a maximum of 5 concurrent tasks to maintain performance. The services will communicate via a gateway, written in Java, which manages routing between them.

# Conclusion

This "Guess the Country by Flag" game demonstrates a use of microservices architecture. By separating user registration and gameplay into different microservices, the system is flexible and scalable.

