### Explanation of the Code

This code is an **API Gateway** for a microservices-based system, built using **Express.js**. The gateway routes requests to different backend services while handling **load balancing, caching with Redis, and logging with Winston and Logstash**.

---

## **1. Dependencies and Setup**
```js
const express = require("express");
const axios = require("axios");
const redis = require("redis");
const winston = require('winston');
```
- **Express.js** – Web framework for handling HTTP requests.
- **Axios** – For making HTTP requests to backend services.
- **Redis** – Used for caching API responses.
- **Winston** – Logging library for logging to Logstash.

```js
require('dotenv').config();
const LOGGING = parseInt(process.env.LOGGING);
const LOGSTASH_HOST = process.env.LOGSTASH_HOST;
const LOGSTASH_PORT = process.env.LOGSTASH_PORT;
```
- Loads environment variables for **logging configuration**.

---

## **2. Logging Setup**
```js
const infoLogger = winston.createLogger({
    transports: [
        new winston.transports.Http({
            host: LOGSTASH_HOST,
            port: LOGSTASH_PORT,
            level: 'info',
        })
    ],
});

const errLogger = winston.createLogger({
    transports: [
        new winston.transports.Http({
            host: LOGSTASH_HOST,
            port: LOGSTASH_PORT,
            level: 'error'
        })
    ]
});
```
- Sets up **two loggers**: one for **info** logs, another for **error** logs.
- Logs are sent to **Logstash** for centralized logging.

```js
function logMsg(msg) {
    if (LOGGING) {
        infoLogger.info(JSON.stringify({
            "service": "gateway",
            "msg": msg
        }));
    }
}

function logErr(msg) {
    if (LOGGING) {
        errLogger.error(JSON.stringify({
            "service": "gateway",
            "error": msg
        }));
    }
}
```
- Helper functions to log messages **only if logging is enabled**.

---

## **3. Redis Caching Setup**
```js
const redisClient = redis.createClient({ url: process.env.REDIS_URL || "redis://gateway-cache:6379" });
redisClient.connect();
```
- Connects to a **Redis cache** (defaulting to `gateway-cache:6379`).
- Used to cache API responses to improve performance.

---

## **4. Load Balancing Setup**
```js
const chatBases = ["http://chat-1:8000", "http://chat-2:8000", "http://chat-3:8000"];
const assetBases = ["http://asset-1:8000", "http://asset-2:8000", "http://asset-3:8000"];
let chatIndex = 0;
let assetIndex = 0;
```
- Defines backend **Chat Service** and **Asset Service** instances.
- Maintains **round-robin load balancing indexes**.

```js
const getNextHost = (serviceHosts, index) => {
    const host = serviceHosts[index % serviceHosts.length];
    return { host, newIndex: (index + 1) % serviceHosts.length };
};
```
- **Round-robin algorithm** to distribute requests across multiple instances.

---

## **5. Caching Mechanism**
```js
const cachingBlacklist = [
    "/chatApp/utilities/status/",
    "/assetApp/utilities/status/",
    "/chats/"
];
```
- **Excludes certain API routes** from caching.

```js
const getCachingKey = (req) => {
    if (
        req.method !== "GET" || 
        cachingBlacklist.some((path) => req.originalUrl.startsWith(path))
    ) {
        return null;
    }

    let cachingKey = `get:${req.originalUrl}`;
    const username = req.headers["x-username"];

    if (username) {
        cachingKey += `:username=${username}`;
    }

    return cachingKey;
};
```
- **Only caches GET requests**.
- **Generates a caching key** based on the request URL and username (if present).

---

## **6. Handling Requests**
```js
const handleRequest = async (req, res, serviceHosts, indexTracker) => {
    const { host, newIndex } = getNextHost(serviceHosts, indexTracker);
    const cachingKey = getCachingKey(req);
```
- Selects a backend **service instance** using round-robin.
- Generates a **caching key** if caching is applicable.

```js
    if (cachingKey) {
        try {
            const cachedResponse = await redisClient.get(cachingKey);
            if (cachedResponse) {
                logMsg(`LOG: Retrieving ${req.method}:${req.originalUrl} from cache`);
                return res.json(JSON.parse(cachedResponse));
            }
        } catch (error) {
            console.error("Redis error:", error);
        }
    }
```
- **Checks Redis cache** before making a backend request.

```js
    logMsg(`LOG: Requesting ${req.method}:${req.originalUrl} from ${host}`);

    const url = `${host}${req.originalUrl}`;
    try {
        const response = await axios({
            method: req.method,
            url: url,
            data: req.body,
            headers: req.headers
        });

        if (cachingKey) {
            await redisClient.setEx(cachingKey, 60, JSON.stringify(response.data));
        }

        res.status(response.status).json(response.data);
    } catch (error) {
        logErr(`ERR: Received ${error.response?.status || 500} response from ${new URL(url).hostname.split(':')[0]}`);

        res.status(error.response?.status || 500).json(error.response?.data || { detail: "Internal Server Error" });
    }
```
- **Makes a request** to the selected backend service.
- **Stores successful responses in Redis** (cache expiration: 60s).
- Logs and returns **errors appropriately**.

```js
    if (serviceHosts === chatBases) {
        chatIndex = newIndex;
    } else {
        assetIndex = newIndex;
    }
};
```
- Updates the **round-robin index** after each request.

---

## **7. Defining API Routes**
```js
const chatEndpoits = [
    "/chats/",
    "/chats/:chat_id/",
    "/chats/unseen-list/",
    "/messages/send-to-chat/:chat_id/",
    "/messages/:message_id/",
    "/messages/:message_id/mark-as-seen/",
    "/chatApp/utilities/status/"
];
const assetEndpoints = [
    "/assets/",
    "/assets/:asset_id/",
    "/assets/:asset_id/add-to-scene/:scene_id/",
    "/assets/list-for-scene/:scene_id/",
    "/assets/get-file-url/:asset_id/",
    "/scenes/",
    "/scenes/:scene_id/",
    "/assetApp/utilities/status/",
];

chatEndpoits.forEach((route) => {
    app.all(route, (req, res) => handleRequest(req, res, chatBases, chatIndex));
});

assetEndpoints.forEach((route) => {
    app.all(route, (req, res) => handleRequest(req, res, assetBases, assetIndex));
});
```
- **Maps API routes** to their corresponding backend services.
- Uses `app.all()` to support **GET, POST, PUT, DELETE** requests.

---

## **8. API Gateway Status Route**
```js
app.get('/status/', (req, res) => {
    res.status(200).json({ message: `API Gateway at 127.0.0.1:${PORT} is up and running` });
});
```
- **Health check** route for the API Gateway.

---

## **9. Start the Server**
```js
const PORT = 8080;
app.listen(PORT, () => {
    console.log(`Gateway running at 127.0.0.1:${PORT}`);
});
```
- Starts the **Express server** on port **8080**.

---

## **Summary**
- **Acts as a gateway** for the Chat and Asset services.
- Implements **round-robin load balancing** across multiple backend instances.
- Uses **Redis caching** to store GET request responses.
- Implements **centralized logging** with Logstash.
- **Handles errors gracefully** while logging failures.

This is a **scalable and efficient API Gateway** that improves performance and reliability. 🚀
